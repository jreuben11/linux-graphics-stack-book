# Chapter 165: Vulkan Video Encode

**Part VII — Application APIs and Middleware**

**Audiences targeted**: Graphics and multimedia application developers integrating hardware video encode into Vulkan pipelines; driver developers implementing `VK_KHR_video_encode_h264`, `VK_KHR_video_encode_h265`, and `VK_KHR_video_encode_av1`; and engineers building zero-copy transcode pipelines, game-capture stacks, and low-latency VR streaming systems.

This chapter is the encode counterpart to **Chapter 50 (Vulkan Video Decode)**. Ch50 covers the shared `VK_KHR_video_queue` infrastructure, decode session lifecycle, H.264/H.265/AV1 decode extensions, DPB management, and FFmpeg decode integration. This chapter focuses exclusively on the encode side: `VK_KHR_video_encode_queue` and the codec-specific encode extensions, rate control, Mesa driver implementations, FFmpeg and GStreamer encode integration, and latency-optimised streaming use cases.

---

## Table of Contents

1. [VK_KHR_video_encode_queue: Encode vs Decode Architecture](#1-vk_khr_video_encode_queue-encode-vs-decode-architecture)
2. [H.264 Encode: VK_KHR_video_encode_h264](#2-h264-encode-vk_khr_video_encode_h264)
3. [H.265/HEVC Encode: VK_KHR_video_encode_h265](#3-h265hevc-encode-vk_khr_video_encode_h265)
4. [AV1 Encode: VK_KHR_video_encode_av1](#4-av1-encode-vk_khr_video_encode_av1)
5. [Rate Control: CBR, VBR, CQP, and Per-Layer Control](#5-rate-control-cbr-vbr-cqp-and-per-layer-control)
6. [Mesa RADV and ANV Encode Implementation](#6-mesa-radv-and-anv-encode-implementation)
7. [FFmpeg Vulkan Encode Integration](#7-ffmpeg-vulkan-encode-integration)
8. [GStreamer Vulkan Encode](#8-gstreamer-vulkan-encode)
9. [Latency-Optimised Encode for Streaming](#9-latency-optimised-encode-for-streaming)
10. [Integrations](#10-integrations)

---

## 1. VK_KHR_video_encode_queue: Encode vs Decode Architecture

`VK_KHR_video_encode_queue` was finalised in December 2023 alongside the H.264 and H.265 encode codec extensions. [Source: Khronos Finalizes Vulkan Video H.264/H.265 Encode](https://www.khronos.org/blog/khronos-finalizes-vulkan-video-extensions-for-accelerated-h.264-and-h.265-encode) It shares the `VK_KHR_video_queue` infrastructure — queue discovery, `VkVideoSessionKHR`, `VkVideoSessionParametersKHR`, format queries, capability queries — but differs from the decode path in several fundamental ways.

### Structural Differences from Decode

**Input and output are reversed.** In decode, the input is a compressed bitstream in a `VkBuffer` and the output is a reconstructed frame in a `VkImage`. In encode, the input is an uncompressed source frame in a `VkImage` (the *input picture resource*) and the output is a compressed bitstream written into a `VkBuffer` (the *destination buffer*).

**The encoder owns reference picture management for temporal prediction.** A decoder receives the reference frame list from the already-encoded bitstream; an encoder must decide which past frames to reference, how many B-frames to insert, when to emit an IDR frame, and how to balance reference quality against encode latency. The application controls these decisions through per-frame `VkVideoEncodeH264PictureInfoKHR` (or H.265/AV1 equivalents) chained onto `VkVideoEncodeInfoKHR`.

**Rate control is an encode-only concern.** The encoder must target a bitrate or quality level by adjusting the quantisation parameter (QP) on a per-frame or per-slice/CTU basis. This is realised through `VkVideoEncodeRateControlInfoKHR` (covered in Section 5).

**Quality levels replace decode profiles.** Implementations expose encoder preset quality levels from 0 to `maxQualityLevels - 1` (from `VkVideoEncodeCapabilitiesKHR.maxQualityLevels`). Quality 0 is the default (fastest/lowest quality); higher indices trade speed for visual quality. Properties for each quality level are retrieved with `vkGetPhysicalDeviceVideoEncodeQualityLevelPropertiesKHR`.

**Encode result queries.** After submitting a `vkCmdEncodeVideoKHR` command buffer, the application must read back how many bytes were actually written to the destination buffer and whether each frame encoded successfully. This uses `VkQueryPool` with `VK_QUERY_TYPE_VIDEO_ENCODE_FEEDBACK_KHR`.

### Encode Queue Discovery

Encode queues are distinct from decode queues. They advertise `VK_QUEUE_VIDEO_ENCODE_BIT_KHR` in their `queueFlags`:

```c
/* src: application encode queue setup */
uint32_t count = 0;
vkGetPhysicalDeviceQueueFamilyProperties2(phys_dev, &count, NULL);

VkQueueFamilyVideoPropertiesKHR *video_props =
    calloc(count, sizeof(*video_props));
VkQueueFamilyProperties2 *props =
    calloc(count, sizeof(*props));

for (uint32_t i = 0; i < count; i++) {
    props[i].sType  = VK_STRUCTURE_TYPE_QUEUE_FAMILY_PROPERTIES_2;
    props[i].pNext  = &video_props[i];
    video_props[i].sType = VK_STRUCTURE_TYPE_QUEUE_FAMILY_VIDEO_PROPERTIES_KHR;
}
vkGetPhysicalDeviceQueueFamilyProperties2(phys_dev, &count, props);

uint32_t encode_qf = UINT32_MAX;
for (uint32_t i = 0; i < count; i++) {
    VkQueueFlags f = props[i].queueFamilyProperties.queueFlags;
    if (f & VK_QUEUE_VIDEO_ENCODE_BIT_KHR) {
        if (video_props[i].videoCodecOperations &
            VK_VIDEO_CODEC_OPERATION_ENCODE_H264_BIT_KHR) {
            encode_qf = i;
            break;
        }
    }
}
```

On AMD hardware with RADV, the encode and decode queues are typically distinct queue families, both backed by the VCN (Video Core Next) engine. On Intel ANV, they may be the same queue family index but with different `videoCodecOperations` bitmask bits.

### Encode Capability Query

Before creating an encode session, query `VkVideoEncodeCapabilitiesKHR` (chained off `VkVideoCapabilitiesKHR`) for limits and feature flags:

```c
/* src: application capability query — VK_KHR_video_encode_queue */
VkVideoEncodeH264CapabilitiesKHR h264_enc_caps = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_CAPABILITIES_KHR,
};
VkVideoEncodeCapabilitiesKHR enc_caps = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_ENCODE_CAPABILITIES_KHR,
    .pNext = &h264_enc_caps,
};
VkVideoCapabilitiesKHR caps = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_CAPABILITIES_KHR,
    .pNext = &enc_caps,
};

VkVideoProfileInfoKHR encode_profile = {
    .sType               = VK_STRUCTURE_TYPE_VIDEO_PROFILE_INFO_KHR,
    .videoCodecOperation = VK_VIDEO_CODEC_OPERATION_ENCODE_H264_BIT_KHR,
    .chromaSubsampling   = VK_VIDEO_CHROMA_SUBSAMPLING_420_BIT_KHR,
    .lumaBitDepth        = VK_VIDEO_COMPONENT_BIT_DEPTH_8_BIT_KHR,
    .chromaBitDepth      = VK_VIDEO_COMPONENT_BIT_DEPTH_8_BIT_KHR,
};
vkGetPhysicalDeviceVideoCapabilitiesKHR(phys_dev, &encode_profile, &caps);

/* caps.maxCodedExtent: maximum resolution supported for this profile */
/* enc_caps.maxQualityLevels: number of quality presets */
/* enc_caps.encodeInputPictureGranularity: input image alignment requirements */
/* enc_caps.supportedEncodeFeedbackFlags: which query results are available */
/* h264_enc_caps.minQp, h264_enc_caps.maxQp: QP range */
/* h264_enc_caps.maxSliceCount: maximum slices per frame */
```

The `encodeInputPictureGranularity` field is significant: it gives the alignment constraint for the input picture extent. An encoder that processes 16x16 macro-blocks (H.264 baseline) requires the input image to be padded to 16-pixel boundaries. The application must create input images with extents rounded up to these granularity multiples.

### vkCmdEncodeVideoKHR: The Core Encode Command

```c
/* Signature — VK_KHR_video_encode_queue */
void vkCmdEncodeVideoKHR(
    VkCommandBuffer              commandBuffer,
    const VkVideoEncodeInfoKHR  *pEncodeInfo);

typedef struct VkVideoEncodeInfoKHR {
    VkStructureType                    sType;
    const void                        *pNext;          /* codec-specific info */
    VkVideoEncodeFlagsKHR              flags;
    VkBuffer                           dstBuffer;       /* compressed output */
    VkDeviceSize                       dstBufferOffset;
    VkDeviceSize                       dstBufferRange;
    VkVideoPictureResourceInfoKHR      srcPictureResource;  /* uncompressed input */
    const VkVideoReferenceSlotInfoKHR *pSetupReferenceSlot; /* DPB activation */
    uint32_t                           referenceSlotCount;
    const VkVideoReferenceSlotInfoKHR *pReferenceSlots;     /* reference frames */
    uint32_t                           precedingExternallyEncodedBytes;
} VkVideoEncodeInfoKHR;
```

The `dstBuffer` must have been created with `VK_BUFFER_USAGE_VIDEO_ENCODE_DST_BIT_KHR`. The encoder writes the compressed NAL units (H.264/H.265) or OBUs (AV1) into the buffer starting at `dstBufferOffset`. The actual number of bytes written is retrieved via the encode feedback query.

### Encode Session Lifecycle and DPB Image Setup

Creating an encode session follows the same steps as decode (Ch50 Section 2) but with encode-specific image usage flags. The encode DPB (Decoded Picture Buffer, here acting as a Reconstructed Picture Buffer for the reference frames the encoder creates) uses `VK_IMAGE_USAGE_VIDEO_ENCODE_DPB_BIT_KHR`, and the source (input) picture uses `VK_IMAGE_USAGE_VIDEO_ENCODE_SRC_BIT_KHR`:

```c
/* src: encode session image setup */
VkVideoSessionCreateInfoKHR enc_session_ci = {
    .sType              = VK_STRUCTURE_TYPE_VIDEO_SESSION_CREATE_INFO_KHR,
    .queueFamilyIndex   = encode_queue_family,
    .pVideoProfile      = &encode_profile,   /* ENCODE_H264_BIT, 420, 8-bit */
    .pictureFormat      = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,  /* NV12 input */
    .maxCodedExtent     = { 1920, 1080 },
    .referencePictureFormat = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,
    .maxDpbSlots        = 4,     /* H.264: max 4 reference frames for streaming */
    .maxActiveReferencePictures = 2,
    .pStdHeaderVersion  = &std_header_version,
};
VkVideoSessionKHR encode_session;
vkCreateVideoSessionKHR(device, &enc_session_ci, NULL, &encode_session);

/* Allocate session memory (same opaque binding model as decode) */
uint32_t mem_req_count;
vkGetVideoSessionMemoryRequirementsKHR(device, encode_session, &mem_req_count, NULL);
/* ... allocate VkDeviceMemory and bind via vkBindVideoSessionMemoryKHR ... */

/* Create DPB images for reconstructed reference pictures */
VkImageCreateInfo dpb_img_ci = {
    .sType       = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .imageType   = VK_IMAGE_TYPE_2D,
    .format      = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,
    .extent      = { 1920, 1088, 1 },   /* height padded to 16-pixel boundary */
    .mipLevels   = 1,
    .arrayLayers = 4,   /* one layer per DPB slot */
    .usage       = VK_IMAGE_USAGE_VIDEO_ENCODE_DPB_BIT_KHR,
};
/* Source/input images for frames to encode: */
VkImageCreateInfo src_img_ci = {
    .sType       = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .format      = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,
    .extent      = { 1920, 1080, 1 },
    .mipLevels   = 1, .arrayLayers = 1,
    .usage       = VK_IMAGE_USAGE_VIDEO_ENCODE_SRC_BIT_KHR
                 | VK_IMAGE_USAGE_TRANSFER_DST_BIT,
};
```

The `encodeInputPictureGranularity` from the capability query dictates the alignment of `codedExtent` in `srcPictureResource`. If granularity is `{16, 16}` (H.264 macroblock size), a 1920×1080 source image must have its `codedExtent` set to `{1920, 1080}` (already aligned), while a 1280×720 image would similarly be fine. A non-aligned resolution like 1280×544 would need `codedExtent` padded to the next multiple.

---

## 2. H.264 Encode: VK_KHR_video_encode_h264

`VK_KHR_video_encode_h264` was ratified in December 2023. [Source: Khronos H.264/H.265 Encode Ratification](https://www.khronos.org/blog/khronos-finalizes-vulkan-video-extensions-for-accelerated-h.264-and-h.265-encode) It provides the structures needed to deliver SPS/PPS header parameters, per-frame slice configuration, and rate control hints to the H.264 hardware encoder.

### Session Parameters: SPS and PPS

H.264 encode requires populating `StdVideoH264SequenceParameterSet` and `StdVideoH264PictureParameterSet` into the session parameters object before encoding begins. These are the same standard-defined structures used in decode, but for encode the application must generate them (or accept driver-generated defaults) rather than parse them from a bitstream:

```c
/* src: H.264 encode — session parameters creation */
StdVideoH264SequenceParameterSet encode_sps = {
    .profile_idc     = STD_VIDEO_H264_PROFILE_IDC_HIGH,
    .level_idc       = STD_VIDEO_H264_LEVEL_IDC_4_1,
    .chroma_format_idc = STD_VIDEO_H264_CHROMA_FORMAT_IDC_420,
    .pic_width_in_mbs_minus1      = (1920 / 16) - 1,
    .pic_height_in_map_units_minus1 = (1080 / 16) - 1,
    .frame_crop_right_offset  = 0,
    .frame_crop_bottom_offset = 0,
    .seq_parameter_set_id     = 0,
    .log2_max_frame_num_minus4 = 4,
    /* ... other SPS fields parsed from intent or queried from driver ... */
};

StdVideoH264PictureParameterSet encode_pps = {
    .seq_parameter_set_id = 0,
    .pic_parameter_set_id = 0,
    .num_ref_idx_l0_default_active_minus1 = 0,
    .num_ref_idx_l1_default_active_minus1 = 0,
    .weighted_bipred_idc = STD_VIDEO_H264_WEIGHTED_BIPRED_IDC_DEFAULT,
    .pic_init_qp_minus26 = 0,
    /* ... */
};

VkVideoEncodeH264SessionParametersAddInfoKHR h264_add = {
    .sType       = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_SESSION_PARAMETERS_ADD_INFO_KHR,
    .stdSPSCount = 1,
    .pStdSPSs    = &encode_sps,
    .stdPPSCount = 1,
    .pStdPPSs    = &encode_pps,
};
VkVideoEncodeH264SessionParametersCreateInfoKHR h264_params_ci = {
    .sType             = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_SESSION_PARAMETERS_CREATE_INFO_KHR,
    .maxStdSPSCount    = 32,
    .maxStdPPSCount    = 256,
    .pParametersAddInfo = &h264_add,
};
VkVideoSessionParametersCreateInfoKHR params_ci = {
    .sType        = VK_STRUCTURE_TYPE_VIDEO_SESSION_PARAMETERS_CREATE_INFO_KHR,
    .pNext        = &h264_params_ci,
    .videoSession = encode_session,
};
VkVideoSessionParametersKHR session_params;
vkCreateVideoSessionParametersKHR(device, &params_ci, NULL, &session_params);
```

### Per-Frame Encode: Slice Types and IDR Insertion

`VkVideoEncodeH264PictureInfoKHR` controls whether a frame is encoded as an I, P, or B frame and sets the NAL unit header fields. The `pStdPictureInfo` pointer carries `StdVideoEncodeH264PictureInfo`, which includes the slice type (`STD_VIDEO_H264_SLICE_TYPE_I`, `_P`, or `_B`) and the `idr_pic_flag` and `is_reference` flags:

```c
/* src: H.264 per-frame encode — IDR frame example */
StdVideoEncodeH264SliceHeader slice_hdr = {
    .slice_type          = STD_VIDEO_H264_SLICE_TYPE_I,
    .cabac_init_idc      = STD_VIDEO_H264_CABAC_INIT_IDC_0,
    .disable_deblocking_filter_idc = STD_VIDEO_H264_DISABLE_DEBLOCKING_FILTER_IDC_DISABLED,
    /* ... */
};
VkVideoEncodeH264NaluSliceInfoKHR slice_info = {
    .sType          = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_NALU_SLICE_INFO_KHR,
    .constantQp     = 0,   /* 0 means rate-control-managed */
    .pStdSliceHeader = &slice_hdr,
};
StdVideoEncodeH264PictureInfo std_pic = {
    .flags = {
        .IdrPicFlag     = 1,   /* force IDR */
        .is_reference   = 1,
    },
    .seq_parameter_set_id = 0,
    .pic_parameter_set_id = 0,
    .pictureType          = STD_VIDEO_H264_PICTURE_TYPE_IDR,
    .frameNum             = 0,
    .PicOrderCnt          = 0,
};
VkVideoEncodeH264PictureInfoKHR h264_enc_pic = {
    .sType               = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_PICTURE_INFO_KHR,
    .naluSliceEntryCount = 1,
    .pNaluSliceEntries   = &slice_info,
    .pStdPictureInfo     = &std_pic,
    .generatePrefixNalu  = VK_FALSE,
};
VkVideoEncodeInfoKHR encode_info = {
    .sType            = VK_STRUCTURE_TYPE_VIDEO_ENCODE_INFO_KHR,
    .pNext            = &h264_enc_pic,
    .dstBuffer        = output_bitstream_buf,
    .dstBufferOffset  = 0,
    .dstBufferRange   = output_buf_size,
    .srcPictureResource = {
        .sType            = VK_STRUCTURE_TYPE_VIDEO_PICTURE_RESOURCE_INFO_KHR,
        .codedOffset      = {0, 0},
        .codedExtent      = {1920, 1080},
        .baseArrayLayer   = 0,
        .imageViewBinding = source_nv12_image_view,
    },
    .pSetupReferenceSlot = &dpb_setup_slot,
    .referenceSlotCount  = 0,   /* IDR has no references */
    .pReferenceSlots     = NULL,
};
vkCmdEncodeVideoKHR(cmd, &encode_info);
```

**IDR insertion**: The application controls IDR insertion by setting `StdVideoEncodeH264PictureInfo.flags.IdrPicFlag = 1`. This should be done at the start of each GOP and on scene-cut detection (Section 9 covers the low-latency streaming pattern). The `frameNum` field must be reset to 0 after each IDR.

**B-frame support**: B-frame capability is reported via the `VK_VIDEO_ENCODE_H264_CAPABILITY_B_FRAME_IN_L1_LIST_BIT_KHR` flag in `VkVideoEncodeH264CapabilitiesKHR.flags`. RDNA3 (VCN 4.x) and Intel Arc (Xe HPG) both support B-frames in H.264 encode. For B-frames, `pReferenceSlots` must include both an L0 (forward) and L1 (backward) reference, and `StdVideoEncodeH264SliceHeader.slice_type` is set to `STD_VIDEO_H264_SLICE_TYPE_B`.

### Rate Control for H.264

`VkVideoEncodeH264RateControlInfoKHR` provides GOP structure parameters to the encoder, chained off `VkVideoEncodeRateControlInfoKHR` in a `vkCmdControlVideoCodingKHR` call:

```c
/* src: H.264 encode rate control setup */
VkVideoEncodeH264RateControlInfoKHR h264_rc = {
    .sType                  = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_RATE_CONTROL_INFO_KHR,
    .flags                  = VK_VIDEO_ENCODE_H264_RATE_CONTROL_REFERENCE_PATTERN_FLAT_BIT_KHR,
    .gopFrameCount          = 120,   /* 4 seconds at 30fps */
    .idrPeriod              = 120,
    .consecutiveBFrameCount = 2,     /* 2 B-frames between I and P */
    .temporalLayerCount     = 1,
};
VkVideoEncodeRateControlLayerInfoKHR rc_layer = {
    .sType                = VK_STRUCTURE_TYPE_VIDEO_ENCODE_RATE_CONTROL_LAYER_INFO_KHR,
    .pNext                = &(VkVideoEncodeH264RateControlLayerInfoKHR){
        .sType     = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_RATE_CONTROL_LAYER_INFO_KHR,
        .useMinQp  = VK_TRUE,
        .minQp     = { .qpI = 18, .qpP = 20, .qpB = 22 },
        .useMaxQp  = VK_TRUE,
        .maxQp     = { .qpI = 36, .qpP = 38, .qpB = 40 },
    },
    .averageBitrate       = 4000000,   /* 4 Mbit/s */
    .maxBitrate           = 6000000,
    .frameRateNumerator   = 30,
    .frameRateDenominator = 1,
};
VkVideoEncodeRateControlInfoKHR rc = {
    .sType               = VK_STRUCTURE_TYPE_VIDEO_ENCODE_RATE_CONTROL_INFO_KHR,
    .pNext               = &h264_rc,
    .rateControlMode     = VK_VIDEO_ENCODE_RATE_CONTROL_MODE_CBR_BIT_KHR,
    .layerCount          = 1,
    .pLayers             = &rc_layer,
    .virtualBufferSizeInMs    = 1000,
    .initialVirtualBufferSizeInMs = 0,
};
VkVideoCodingControlInfoKHR control = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_CODING_CONTROL_INFO_KHR,
    .flags = VK_VIDEO_CODING_CONTROL_ENCODE_RATE_CONTROL_BIT_KHR,
    .pNext = &rc,
};
vkCmdControlVideoCodingKHR(cmd, &control);
```

---

## 3. H.265/HEVC Encode: VK_KHR_video_encode_h265

`VK_KHR_video_encode_h265` was also ratified in December 2023 and follows the same structural pattern as H.264 encode but incorporates the HEVC three-parameter-set model (VPS, SPS, PPS) and its tile/CTU (Coding Tree Unit) concepts.

### Session Parameters: VPS, SPS, PPS

H.265 introduces the Video Parameter Set (VPS), which describes stream-level properties such as the PTL (Profile, Tier, Level) record. The encode session parameters object must include VPS, SPS, and PPS records:

```c
/* src: H.265 encode session parameters */
StdVideoH265VideoParameterSet encode_vps = {
    .vps_video_parameter_set_id = 0,
    .vps_max_sub_layers_minus1  = 0,
    /* pDecPicBufMgr, pHrdParameters, pProfileTierLevel populated per content */
};
StdVideoH265SequenceParameterSet encode_sps = {
    .sps_video_parameter_set_id = 0,
    .sps_seq_parameter_set_id   = 0,
    .chroma_format_idc          = STD_VIDEO_H265_CHROMA_FORMAT_IDC_420,
    .pic_width_in_luma_samples  = 1920,
    .pic_height_in_luma_samples = 1080,
    .log2_min_luma_coding_block_size_minus3 = 0,   /* min CB = 8x8 */
    .log2_diff_max_min_luma_coding_block_size = 2,  /* max CB = 64x64 CTU */
    /* ... */
};

VkVideoEncodeH265SessionParametersAddInfoKHR h265_add = {
    .sType       = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H265_SESSION_PARAMETERS_ADD_INFO_KHR,
    .stdVPSCount = 1,
    .pStdVPSs    = &encode_vps,
    .stdSPSCount = 1,
    .pStdSPSs    = &encode_sps,
    .stdPPSCount = 1,
    .pStdPPSs    = &encode_pps,
};
VkVideoEncodeH265SessionParametersCreateInfoKHR h265_params_ci = {
    .sType             = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H265_SESSION_PARAMETERS_CREATE_INFO_KHR,
    .maxStdVPSCount    = 1,
    .maxStdSPSCount    = 1,
    .maxStdPPSCount    = 1,
    .pParametersAddInfo = &h265_add,
};
```

### CTU Structure and Tile Encoding

H.265 organises the picture into Coding Tree Units (CTUs), which function as the analogue of H.264 macro-blocks but with a quadtree split structure and sizes up to 64×64 luma samples. The `log2_diff_max_min_luma_coding_block_size` in the SPS determines the maximum CTU size the encoder will use.

H.265 also introduces tiles: rectangular regions of CTUs that can be encoded independently. This is architecturally important because tiles enable parallel encode across multiple hardware threads or CTU rows. `VkVideoEncodeH265CapabilitiesKHR` exposes `maxTiles` (the maximum tile row/column count the implementation supports) and the `VK_VIDEO_ENCODE_H265_CAPABILITY_MULTIPLE_TILES_PER_SLICE_BIT_KHR` flag indicating whether multiple tiles can share a single slice NAL unit.

Per-frame encode uses `VkVideoEncodeH265PictureInfoKHR` with `pNaluSliceSegmentEntries` listing each slice segment (H.265 uses slice segments rather than H.264 slices):

```c
/* src: H.265 per-frame encode */
VkVideoEncodeH265NaluSliceSegmentInfoKHR seg = {
    .sType            = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H265_NALU_SLICE_SEGMENT_INFO_KHR,
    .constantQp       = 0,
    .pStdSliceSegmentHeader = &std_slice_hdr,
};
VkVideoEncodeH265PictureInfoKHR h265_enc_pic = {
    .sType                    = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H265_PICTURE_INFO_KHR,
    .naluSliceSegmentEntryCount = 1,
    .pNaluSliceSegmentEntries   = &seg,
    .pStdPictureInfo            = &std_h265_pic_info,
};
```

### Bandwidth Advantage

H.265 achieves roughly equivalent visual quality to H.264 at approximately half the bitrate for the same resolution, owing to its larger CTU sizes, improved intra prediction modes, and motion estimation over larger areas. [Source: HEVC vs H.264 compression analysis](https://www.streamingmedia.com/Articles/Editorial/Featured-Articles/HEVC-vs-H.264-One-Year-Later-104124.aspx) The lookahead buffer — the number of future frames the encoder inspects before committing to a QP and reference structure decision — is not explicitly modelled in the Vulkan Video spec, but hardware implementations such as AMD VCN and Intel VDENC implement their own lookahead strategies that are activated through the quality preset level parameter.

H.265 encode is available on RDNA 2 and later (VCN 3.x+) and on Intel Xe HPG (Arc) and newer via ANV.

---

## 4. AV1 Encode: VK_KHR_video_encode_av1

`VK_KHR_video_encode_av1` was published in November 2024 as part of Vulkan 1.3.302 and represents the most recent addition to the Vulkan Video encode family. [Source: VK_KHR_video_encode_av1 proposal](https://docs.vulkan.org/features/latest/features/proposals/VK_KHR_video_encode_av1.html) AV1 encode is architecturally more complex than H.264/H.265 because of its larger superblock sizes (64×64 or 128×128), its open-coded tile structure, and its more flexible reference frame model.

### Session Parameters: Sequence Header OBU

AV1 encode stores a single `StdVideoAV1SequenceHeader` in the session parameters object. Unlike H.264's multiple SPS/PPS pairs, AV1 uses one sequence header per session; the application creates a new `VkVideoSessionParametersKHR` when the sequence header changes:

```c
/* src: AV1 encode session parameters */
StdVideoAV1SequenceHeader av1_seq_hdr = {
    .flags = {
        .still_picture              = 0,
        .reduced_still_picture_hdr  = 0,
        .enable_filter_intra        = 1,
        .enable_intra_edge_filter   = 1,
        .enable_interintra_compound = 1,
        .enable_masked_compound     = 1,
    },
    .seq_profile = STD_VIDEO_AV1_PROFILE_MAIN,
    .frame_width_bits_minus_1  = 11,  /* widths up to 4096 */
    .frame_height_bits_minus_1 = 11,
    .max_frame_width_minus_1   = 1919,
    .max_frame_height_minus_1  = 1079,
    /* ... color_config, timing_info, film_grain_params_present etc. */
};
VkVideoEncodeAV1SessionParametersCreateInfoKHR av1_params_ci = {
    .sType             = VK_STRUCTURE_TYPE_VIDEO_ENCODE_AV1_SESSION_PARAMETERS_CREATE_INFO_KHR,
    .pStdSequenceHeader = &av1_seq_hdr,
};
```

### Frame Types and Prediction Modes

`VkVideoEncodeAV1PictureInfoKHR` controls the AV1 frame type and the prediction mode. The extension maps the complex AV1 frame type / reference frame update semantics onto four prediction modes:

| `VkVideoEncodeAV1PredictionModeKHR` | AV1 frame types | Description |
|-------------------------------------|----------------|-------------|
| `INTRA_ONLY_KHR` | `KEY_FRAME`, `INTRA_ONLY_FRAME` | No inter prediction |
| `SINGLE_REFERENCE_KHR` | `INTER_FRAME`, `SWITCH_FRAME` | One forward reference |
| `UNIDIRECTIONAL_COMPOUND_KHR` | `INTER_FRAME`, `SWITCH_FRAME` | Two forward references |
| `BIDIRECTIONAL_COMPOUND_KHR` | `INTER_FRAME`, `SWITCH_FRAME` | Forward + backward ref |

For streaming applications that require low latency, `SINGLE_REFERENCE_KHR` with `KEY_FRAME` at each IDR interval is the common choice. For film encoding, `BIDIRECTIONAL_COMPOUND_KHR` with a GOP structure analogous to H.264's B-frames yields the highest compression efficiency.

The `referenceNameSlotIndices` field in `VkVideoEncodeAV1PictureInfoKHR` maps each of the seven AV1 reference frame names (`LAST_FRAME` through `ALTREF_FRAME`) to a DPB slot index, mirroring the AV1 decode extension's reference frame model:

```c
/* src: AV1 per-frame encode — inter frame with single reference */
StdVideoEncodeAV1PictureInfo std_av1_pic = {
    .flags = {
        .show_frame    = 1,
        .showable_frame = 1,
        .error_resilient_mode = 0,
    },
    .frame_type        = STD_VIDEO_AV1_FRAME_TYPE_INTER,
    .current_frame_id  = frame_counter % (1 << order_hint_bits),
    /* primary_ref_frame, refresh_frame_flags, etc. */
};
VkVideoEncodeAV1PictureInfoKHR av1_enc_pic = {
    .sType          = VK_STRUCTURE_TYPE_VIDEO_ENCODE_AV1_PICTURE_INFO_KHR,
    .predictionMode = VK_VIDEO_ENCODE_AV1_PREDICTION_MODE_SINGLE_REFERENCE_KHR,
    .rateControlGroup = VK_VIDEO_ENCODE_AV1_RATE_CONTROL_GROUP_PREDICTIVE_KHR,
    .constantQIndex = 0,   /* 0 = rate-control managed */
    .pStdPictureInfo = &std_av1_pic,
    /* referenceNameSlotIndices[0] = LAST_FRAME DPB slot */
    .referenceNameSlotIndices = { dpb_last_slot, -1, -1, -1, -1, -1, -1 },
};
```

### Tile Encoding

AV1's tile structure divides the frame into a grid of independently decodable rectangular regions. `VkVideoEncodeAV1CapabilitiesKHR` reports:

- `maxTiles`: maximum `VkExtent2D` in tile rows/columns (e.g., `{4, 4}` = up to 16 tiles)
- `minTileSize` / `maxTileSize`: minimum and maximum tile dimensions in luma samples
- `superblockSizes`: bitmask of supported superblock sizes (`VK_VIDEO_ENCODE_AV1_SUPERBLOCK_SIZE_64_BIT_KHR` and/or `VK_VIDEO_ENCODE_AV1_SUPERBLOCK_SIZE_128_BIT_KHR`)

Larger tile counts enable higher-throughput encoding by allowing the VCN or equivalent engine to process multiple tiles in parallel at the cost of slightly reduced compression efficiency (inter-tile prediction is not possible). For a 1920×1080 encode at the VCN4 superblock size of 64×64, the frame is 30×17 superblocks; a 4×4 tile grid gives 16 tiles of approximately 8×4 superblocks each.

The application configures tile layout through `StdVideoEncodeAV1TileInfoKHR` (carried in the codec standard headers), specifying `tile_cols_log2` and `tile_rows_log2` fields. The AV1 specification requires that tile boundaries align to superblock boundaries, so the encoder rounds the tile dimensions up accordingly. Implementations that report `VK_VIDEO_ENCODE_AV1_CAPABILITY_PER_TILE_QUANTIZATION_BIT_KHR` in `VkVideoEncodeAV1CapabilitiesKHR.flags` allow the application to set a different base quantiser index per tile, enabling spatially adaptive quality — useful for 360° video where the viewing-direction tiles receive higher quality than peripheral tiles.

### Encode Feedback and Bitstream Structure

Unlike H.264/H.265 which produce byte-aligned NAL units, AV1 produces OBUs (Open Bitstream Units). The Vulkan AV1 encode extension outputs one or more Frame OBUs per `vkCmdEncodeVideoKHR` call. If `VK_VIDEO_ENCODE_AV1_CAPABILITY_GENERATE_OBU_EXTENSION_HEADER_BIT_KHR` is supported, the driver inserts OBU extension headers with temporal and spatial layer identifiers. The application must prepend Sequence Header OBUs (from `VkVideoEncodeAV1SessionParametersGetInfoKHR`) at IDR-equivalent points (KEY_FRAME or first INTRA_ONLY_FRAME of a stream).

### Hardware Support Matrix (June 2026)

| Hardware | Driver | AV1 Encode Support |
|----------|--------|--------------------|
| RDNA3 (Navi 3x) | RADV Mesa 25.2+ | VK_KHR_video_encode_av1 (VCN 4.x) |
| RDNA4 (Navi 4x) | RADV Mesa 25.0+ | VK_KHR_video_encode_av1 (VCN 5.x) |
| Intel Arc (Xe HPG, Xe2) | ANV Mesa | Note: AV1 encode status under review |
| NVIDIA RTX 4000 series | Proprietary | VK_KHR_video_encode_av1 (NVENC AV1) |
| NVIDIA RTX 4000 series | NVK Mesa | Note: NVK video encode status needs verification |
| RDNA2 and older | RADV | AV1 encode not supported (VCN 3.x lacks AV1 encode HW) |

> **Note: needs verification** Intel ANV AV1 encode and NVK video encode status are under active development as of mid-2026; consult Mesa release notes for the current state.

---

## 5. Rate Control: CBR, VBR, CQP, and Per-Layer Control

Rate control is the mechanism by which an encoder varies QP (and thus bitrate) across frames to achieve a desired average bitrate or quality level. `VkVideoEncodeRateControlInfoKHR` is the central structure, applied via `vkCmdControlVideoCodingKHR` at the start of a session (or when parameters change):

```c
/* Signature: VK_KHR_video_encode_queue */
typedef struct VkVideoEncodeRateControlInfoKHR {
    VkStructureType                            sType;
    const void                                *pNext;           /* codec RC info */
    VkVideoEncodeRateControlFlagsKHR           flags;
    VkVideoEncodeRateControlModeFlagBitsKHR    rateControlMode;
    uint32_t                                   layerCount;
    const VkVideoEncodeRateControlLayerInfoKHR *pLayers;
    uint32_t                                   virtualBufferSizeInMs;
    uint32_t                                   initialVirtualBufferSizeInMs;
} VkVideoEncodeRateControlInfoKHR;
```

### Rate Control Modes

`VkVideoEncodeRateControlModeFlagBitsKHR` defines four modes:

**`VK_VIDEO_ENCODE_RATE_CONTROL_MODE_DEFAULT_KHR`** — The implementation selects its own rate control algorithm. Equivalent to passing no rate control configuration. Implementations may use VBR, CBR, or a proprietary algorithm.

**`VK_VIDEO_ENCODE_RATE_CONTROL_MODE_DISABLED_BIT_KHR`** — No rate control. The application specifies `constantQp` (H.264/H.265) or `constantQIndex` (AV1) on a per-frame or per-slice basis. This is the Constant QP (CQP) mode. CQP produces consistent visual quality at the cost of unpredictable bitrate, which makes it suitable for archiving but problematic for streaming over a fixed-bandwidth channel.

**`VK_VIDEO_ENCODE_RATE_CONTROL_MODE_CBR_BIT_KHR`** — Constant Bitrate. The implementation adjusts QP frame-to-frame to maintain `averageBitrate` within the virtual buffer window defined by `virtualBufferSizeInMs`. CBR is essential for streaming over network channels with strict bandwidth limits (RTP/UDP, RTSP, WebRTC). The `virtualBufferSizeInMs` parameter functions as the VBV (Video Buffering Verifier) buffer: a larger value allows more latitude but increases end-to-end latency.

**`VK_VIDEO_ENCODE_RATE_CONTROL_MODE_VBR_BIT_KHR`** — Variable Bitrate. The implementation targets `averageBitrate` but may burst up to `maxBitrate` on complex content (fast motion, scene changes) and use lower bitrate on simple content (static backgrounds). VBR produces better visual quality than CBR at the same average bitrate and is preferred for file encoding.

The choice between CBR and VBR has significant practical implications:

- **CBR** is required for live streaming to network endpoints (RTMP, SRT, RTP) where downstream buffers are small. The `virtualBufferSizeInMs` setting controls the smoothing window: 1000 ms is standard for broadcast; 100–200 ms for interactive/gaming streaming.
- **VBR** is preferred for file encoding, where quality is paramount and bitrate variation is acceptable. The `maxBitrate` cap prevents uncontrolled bitrate spikes on scene changes.
- **CQP** (`RATE_CONTROL_MODE_DISABLED`) is used when the application manages bitrate externally or when producing visually lossless encodes for post-production. With CQP, the `constantQp` / `constantQIndex` field in each frame's codec-specific info structure sets the QP directly.

The rate control mode is applied at session initialisation via `vkCmdControlVideoCodingKHR` with the `VK_VIDEO_CODING_CONTROL_ENCODE_RATE_CONTROL_BIT_KHR` flag, and can be changed mid-stream for adaptive streaming scenarios (e.g., switching from VBR to CBR when network conditions degrade).

### Per-Layer Rate Control

H.264 and H.265 both support temporal scalability (SVC-T), where the base temporal layer (TL0) contains I and P frames that can be decoded independently, and higher temporal layers (TL1, TL2) contain B-frames that enhance frame rate but can be dropped to reduce bandwidth. `VkVideoEncodeRateControlInfoKHR.layerCount` specifies the number of temporal layers, and `pLayers` provides per-layer bitrate targets.

For H.264 temporal scalability with two temporal layers:

```c
/* src: H.264 two-layer temporal scalability rate control */
VkVideoEncodeH264RateControlLayerInfoKHR h264_rc_layers[2] = {
    {   /* TL0: base layer — I and P frames */
        .sType    = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_RATE_CONTROL_LAYER_INFO_KHR,
        .useMinQp = VK_TRUE,
        .minQp    = { .qpI = 20, .qpP = 22, .qpB = 0 },
        .useMaxQp = VK_TRUE,
        .maxQp    = { .qpI = 36, .qpP = 36, .qpB = 0 },
    },
    {   /* TL1: enhancement layer — B frames */
        .sType    = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_RATE_CONTROL_LAYER_INFO_KHR,
        .useMinQp = VK_TRUE,
        .minQp    = { .qpI = 0, .qpP = 0, .qpB = 24 },
        .useMaxQp = VK_TRUE,
        .maxQp    = { .qpI = 0, .qpP = 0, .qpB = 40 },
    },
};
VkVideoEncodeRateControlLayerInfoKHR rc_layers[2] = {
    { /* TL0 */ .averageBitrate = 2000000, .maxBitrate = 3000000,
      .frameRateNumerator = 30, .frameRateDenominator = 1,
      .pNext = &h264_rc_layers[0] },
    { /* TL1 */ .averageBitrate = 2000000, .maxBitrate = 4000000,
      .frameRateNumerator = 60, .frameRateDenominator = 1,
      .pNext = &h264_rc_layers[1] },
};
```

Per-layer rate control is particularly useful in adaptive bitrate streaming (ABR) systems where a receiver can drop the enhancement temporal layer to halve the frame rate without full re-encode.

### Driver Rate Control Implementation

The rate control algorithm runs in hardware firmware, not in the host CPU driver:

**AMD VCN** (Video Core Next): Rate control is implemented in VCN firmware loaded via `linux-firmware.git` (`/lib/firmware/amdgpu/vcn_*.bin`). The firmware implements a PID-like controller that monitors the virtual buffer occupancy and adjusts the QP scale map frame-by-frame. The Mesa RADV driver submits rate control parameters as part of the IB (Indirect Buffer) passed to VCN; the firmware interprets these and makes per-frame QP decisions autonomously. [Source: AMD VCN firmware in linux-firmware](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/amdgpu)

**Intel VDENC** (Video Decode and Encode): Intel's Video Decode Box (VD-BOX) contains a VDENC unit that implements CQM (Constant QMatrix) and BRC (Bitrate Rate Control) in hardware. The MFC (Multi Format Codec) pipeline submits media commands (MFX_AVC_IMG_STATE, VDENC_PIPE_BUF_ADDR_STATE, etc.) to the hardware, which includes BRC PAK statistics feedback after each frame. The BRC firmware counter reads the PAK output size and adjusts QP for the next frame. [Source: Intel Media Driver VDENC](https://github.com/intel/media-driver/tree/master/media_driver/agnostic/common/vp)

**NVK/NVENC** (NVIDIA): NVIDIA's NVENC engine implements rate control entirely in a dedicated on-die microcontroller. The Vulkan Video path via NVK (the open-source Mesa NVIDIA driver) is still developing; the proprietary driver's NVENC exposes all rate control modes through its own API.

---

## 6. Mesa RADV and ANV Encode Implementation

### RADV: AMD VCN Encode Pipeline

The RADV implementation of Vulkan Video encode lives in `src/amd/vulkan/radv_video.c` and the shared VCN encode library under `src/amd/common/ac_vcn_enc.c`. [Source: Mesa GitLab — radv video](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/vulkan/video)

When `vkCmdEncodeVideoKHR` is called, RADV:

1. **Selects the VCN encode queue ring** — the AMDGPU kernel driver exposes the VCN engine as a separate ring (`AMDGPU_HW_IP_VCN_ENC`), distinct from the graphics GFX ring and the compute ring.

2. **Constructs the IB (Indirect Buffer)** — RADV builds a sequence of VCN firmware commands: `RENCODE_IB_OP_INITIALIZE` (once per session), `RENCODE_IB_OP_ENCODE` (per frame), and `RENCODE_IB_OP_SUBMIT_ENCODE_HEADERS` (for AV1 OBU header emission).

3. **Emits codec-specific headers** — For AV1 encode, RADV must construct the Sequence Header OBU and Frame Header OBU in the output buffer before VCN writes the tile group data. The AV1 OBU construction logic is in `src/amd/common/ac_vcn_enc_av1.c`.

4. **Manages DPB resources** — RADV tracks which VCN DPB slots map to which Vulkan `VkVideoReferenceSlotInfoKHR` indices and programs the VCN firmware's reference picture buffer addresses accordingly.

5. **Signals completion** — VCN signals a timeline semaphore (or fence) on the AMDGPU kernel side when the encode job completes. RADV translates this into the Vulkan synchronisation primitives the application is waiting on.

For H.264 and H.265 encode, this path has been available and enabled by default on VCN 2.x and 3.x (RDNA 1 and 2) since Mesa 25.0. [Source: Mesa 25.0 release notes](https://docs.mesa3d.org/relnotes/25.0.0.html) AV1 encode via `VK_KHR_video_encode_av1` was merged into Mesa 25.2 for VCN 4.x (RDNA3) hardware, with VCN 5.x (RDNA4) support following in Mesa 25.0 development. [Source: Mesa 25.2 RADV AV1 encode](https://www.phoronix.com/news/RADV-Merges-AV1-Encode)

The encode implementation is exercised via Mesa's video conformance test suite:

```bash
# Run Vulkan Video encode CTS on RADV
deqp-vk -n dEQP-VK.video.encode.h264 --deqp-log-filename=h264_enc.qpa
deqp-vk -n dEQP-VK.video.encode.h265 --deqp-log-filename=h265_enc.qpa
deqp-vk -n dEQP-VK.video.encode.av1  --deqp-log-filename=av1_enc.qpa

# Enable encode on VCN hardware that requires opt-in:
RADV_PERFTEST=video_encode vulkaninfo | grep -i "encode"
```

Mesa 26.1 extended RADV encode with variable slice mode (allowing the application to control slice counts per-frame dynamically), low-latency flags support, and quality level-based encode presets. [Source: Mesa 26.1 release notes](https://docs.mesa3d.org/relnotes/26.1.0.html)

### ANV: Intel VDENC Encode Pipeline

The Intel ANV encode implementation uses the VDENC and MFX fixed-function engine on Gen9+ hardware. The primary encode code is in `src/intel/vulkan/anv_video.c`.

For H.264 encode, ANV constructs an MFX command sequence:

```c
/* src/intel/vulkan/anv_video.c — simplified H.264 encode dispatch */
/* VDENC mode uses a pipeline flush then: */
anv_batch_emit(&cmd->batch, GENX(VDENC_PIPE_MODE_SELECT), mode) {
    mode.StandardSelect      = SS_AVC;
    mode.FrameStatisticsStreamOutputEnable = true;
}
anv_batch_emit(&cmd->batch, GENX(VDENC_SRC_SURFACE_STATE), src) {
    src.Width  = enc_info->srcPictureResource.codedExtent.width;
    src.Height = enc_info->srcPictureResource.codedExtent.height;
    /* ... tiling, format */
}
anv_batch_emit(&cmd->batch, GENX(MFX_AVC_IMG_STATE), img) {
    img.FrameWidth  = coded_width - 1;
    img.FrameHeight = coded_height - 1;
    img.ImageStructure = img_structure;
    /* ... slice control, BRC parameters */
}
anv_batch_emit(&cmd->batch, GENX(VDENC_WALKER_STATE), walker);
anv_batch_emit(&cmd->batch, GENX(MFX_PAK_OBJECT), pak);
```

H.264 and H.265 encode support in ANV was under active development in 2024. As noted in the Mesa 25.3.5 release, ANV encode was temporarily disabled while a correctness issue was investigated; check current Mesa release notes for the latest status. [Source: Mesa 25.3.5 ANV encode status](https://en.linuxadictos.com/table-25-3-5-reinforces-vulkan-video-in-radv-corrects-h-264-reference-management-and-temporarily-disables-video-encoding-in-intel-anv/)

> **Note: needs verification** ANV H.265 and AV1 encode availability; the Intel encoder MR status was evolving rapidly in 2025–2026 and may differ from what is described above. Consult `src/intel/vulkan/anv_video.c` in the current Mesa tree.

---

## 7. FFmpeg Vulkan Encode Integration

FFmpeg added H.264 and H.265/HEVC Vulkan hardware encoders in FFmpeg 7.1 (September 2024), building on the existing Vulkan hwaccel decode framework described in Chapter 50. [Source: FFmpeg 7.1 Vulkan hardware encoding](https://alternativeto.net/news/2024/9/ffmpeg-7-1-released-with-stable-vvc-decoder-vulkan-hardware-encoding-and-much-more/)

### Encoder Names and Hardware Context

The Vulkan encode path reuses the same `AVVulkanDeviceContext` / `AVVulkanFramesContext` / `AVVkFrame` hierarchy as decode (Ch50 Section 7). The encoder names are:

- `h264_vulkan` — H.264 encode via `VK_KHR_video_encode_h264`
- `hevc_vulkan` — H.265 encode via `VK_KHR_video_encode_h265`
- `av1_vulkan` — AV1 encode via `VK_KHR_video_encode_av1` (FFmpeg 8.0+)

To encode, the application creates a device context pointing to the Vulkan instance and passes frames in `AV_PIX_FMT_VULKAN` format (already on the GPU) or `AV_PIX_FMT_NV12` (which FFmpeg will upload via `vulkanupload`):

```c
/* src: FFmpeg Vulkan encode initialisation */
AVBufferRef *hw_device_ctx;
av_hwdevice_ctx_create(&hw_device_ctx, AV_HWDEVICE_TYPE_VULKAN,
                       NULL,   /* device name / DRI path, NULL = first device */
                       NULL, 0);

AVCodec *encoder = avcodec_find_encoder_by_name("h264_vulkan");
AVCodecContext *enc_ctx = avcodec_alloc_context3(encoder);

enc_ctx->width   = 1920;
enc_ctx->height  = 1080;
enc_ctx->pix_fmt = AV_PIX_FMT_VULKAN;   /* GPU-resident input */
enc_ctx->bit_rate = 4000000;
enc_ctx->rc_max_rate = 6000000;
enc_ctx->time_base  = (AVRational){1, 30};
enc_ctx->framerate  = (AVRational){30, 1};
enc_ctx->gop_size   = 120;
enc_ctx->max_b_frames = 2;

/* Associate the hardware device */
AVBufferRef *hw_frames_ctx = av_hwframe_ctx_alloc(hw_device_ctx);
AVHWFramesContext *frames_ctx = (AVHWFramesContext *)hw_frames_ctx->data;
frames_ctx->format    = AV_PIX_FMT_VULKAN;
frames_ctx->sw_format = AV_PIX_FMT_NV12;
frames_ctx->width     = 1920;
frames_ctx->height    = 1080;
av_hwframe_ctx_init(hw_frames_ctx);

enc_ctx->hw_frames_ctx = av_buffer_ref(hw_frames_ctx);
avcodec_open2(enc_ctx, encoder, NULL);
```

### Zero-Copy Decode-to-Encode Transcode

The primary use case for Vulkan Video encode in FFmpeg is zero-copy transcode: the decoded `VkImage` from a decode session is directly consumed as the encode input picture, without any download to system memory. The transcode pipeline on the GPU looks like:

```
H.264 bitstream (VkBuffer)
  → vkCmdDecodeVideoKHR   → decoded NV12 (VkImage, VK_IMAGE_LAYOUT_VIDEO_DECODE_DST_KHR)
  → Pipeline barrier        (VIDEO_DECODE → VIDEO_ENCODE)
  → vkCmdEncodeVideoKHR   → H.265 / AV1 bitstream (VkBuffer)
```

FFmpeg expresses this with its filter graph:

```bash
# Zero-copy H.264 → H.265 transcode staying entirely on GPU
ffmpeg \
  -hwaccel vulkan \
  -hwaccel_output_format vulkan \
  -i input_h264.mp4 \
  -c:v hevc_vulkan \
  -b:v 4M \
  -rc_mode cbr \
  -maxrate 6M \
  output_h265.mp4

# Zero-copy H.264 → AV1 transcode (FFmpeg 8.0+ with av1_vulkan)
ffmpeg \
  -hwaccel vulkan \
  -hwaccel_output_format vulkan \
  -i input_h264.mp4 \
  -c:v av1_vulkan \
  -b:v 2M \
  output_av1.mp4
```

The `-hwaccel_output_format vulkan` flag ensures FFmpeg keeps decoded `AVFrame` objects as `AV_PIX_FMT_VULKAN` through the filter chain. The encode path then receives a `VkImage`-backed frame; the encoder records an image layout transition barrier from `VK_IMAGE_LAYOUT_VIDEO_DECODE_DST_KHR` to `VK_IMAGE_LAYOUT_VIDEO_ENCODE_SRC_KHR` before calling `vkCmdEncodeVideoKHR`.

Importantly, the DMA-BUF / `VK_KHR_external_memory_fd` path that VA-API requires for cross-API surface sharing is entirely avoided. Both decode and encode share the same `VkDevice` and the same image lives in device-local memory throughout the pipeline. [Source: H.264/H.265 Vulkan Encoder Merged into FFmpeg](https://www.phoronix.com/news/FFmpeg-Vulkan-Encode-H.265)

### Encode Result Feedback

After each encode submission, FFmpeg reads back the actual number of bytes written via `VK_QUERY_TYPE_VIDEO_ENCODE_FEEDBACK_KHR`:

```c
/* src: FFmpeg internal — vulkan_encode.c encode feedback query */
VkQueryPoolVideoEncodeFeedbackCreateInfoKHR feedback_ci = {
    .sType = VK_STRUCTURE_TYPE_QUERY_POOL_VIDEO_ENCODE_FEEDBACK_CREATE_INFO_KHR,
    .encodeFeedbackFlags =
        VK_VIDEO_ENCODE_FEEDBACK_BITSTREAM_BUFFER_OFFSET_BIT_KHR |
        VK_VIDEO_ENCODE_FEEDBACK_BITSTREAM_BYTES_WRITTEN_BIT_KHR,
};
/* query result: [offset, bytes_written] per frame */
```

---

## 8. GStreamer Vulkan Encode

GStreamer's Vulkan plugin (`gst-plugins-bad`) includes Vulkan Video encode elements built on top of the `GstVulkanEncoder` base class introduced in GStreamer 1.24. [Source: Vulkan Video Encoder in GStreamer — Igalia](https://blogs.igalia.com/scerveau/vulkan-video-encoder-in-gstreamer/)

### Element Names

- `vulkah264enc` — H.264 encode (GStreamer 1.24+)
- `vulkanh265enc` — H.265 encode (GStreamer 1.26+)
- `vulkav1enc` — AV1 encode (GStreamer 1.28+)
- `vulkanupload` — uploads `GstBuffer` from system memory to GPU (Vulkan) memory, required before encode elements when the source is a CPU element

GStreamer 1.28 (released 2025) expanded Vulkan Video coverage with AV1 and VP9 decode alongside H.264 encode stabilisation. [Source: GStreamer 1.28 Vulkan Video AV1 and H.264 encode](https://linuxiac.com/gstreamer-1-28-multimedia-framework-released/)

### GstVulkanEncoder Base Class

`GstVulkanEncoder` (in `gst-plugins-bad/gst-libs/gst/vulkan/gstvulkanencoder.c`) encapsulates the Vulkan Video encode session lifecycle:

- Creates and manages `VkVideoSessionKHR` and `VkVideoSessionParametersKHR`
- Maintains the DPB image pool and slot-to-frame mapping
- Constructs and submits `vkCmdBeginVideoCodingKHR`, `vkCmdEncodeVideoKHR`, `vkCmdEndVideoCodingKHR` command buffers
- Handles `GstVulkanOperation` for command buffer synchronisation (fences and semaphores)
- Reads encode feedback via `VK_QUERY_TYPE_VIDEO_ENCODE_FEEDBACK_KHR`

Concrete subclasses — `GstVulkanH264Enc`, `GstVulkanH265Enc`, `GstVulkanAV1Enc` — specialise the base class for codec-specific parameter set construction and per-frame structure filling.

### Pipeline Examples

A simple encode pipeline from a test video source:

```bash
# H.264 encode to file via Vulkan
gst-launch-1.0 \
  videotestsrc num-buffers=300 ! video/x-raw,format=NV12,width=1920,height=1080 ! \
  vulkanupload ! \
  vulkah264enc ! \
  h264parse ! \
  mp4mux ! \
  filesink location=output.mp4

# H.265 encode — vulkanupload uploads CPU NV12 to Vulkan image memory
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! video/x-raw,format=NV12 ! \
  vulkanupload ! \
  vulkanh265enc bitrate=4000 ! \
  h265parse ! \
  matroskamux ! \
  filesink location=capture.mkv

# Zero-copy decode → Vulkan encode pipeline (no CPU frames)
gst-launch-1.0 \
  filesrc location=input.mp4 ! qtdemux ! h264parse ! \
  vulkah264dec ! \
  vulkah265enc bitrate=2000 ! \
  h265parse ! filesink location=output_hevc.mkv
```

The `vulkah264dec ! vulkah265enc` chain demonstrates zero-copy GPU-resident transcode: the decoder and encoder share the same `VkDevice` and the decoded `VkImage` is consumed directly by the encoder without touching system memory.

### Build Configuration

Vulkan Video encode elements require the `gst-plugins-bad` Vulkan video build option:

```bash
# Build gst-plugins-bad with Vulkan Video encode support
meson setup build -Dgst-plugins-bad:vulkan-video=enabled
ninja -C build
```

The Vulkan SDK version must be at least 1.3.274 for encode extension support.

---

## 9. Latency-Optimised Encode for Streaming

Low-latency encode has different requirements from archival or broadcast encode: minimising end-to-end latency takes priority over compression efficiency. This section covers the Vulkan Video parameters for low-latency encoding and their use in OBS Studio and wireless VR streaming.

### Low-Latency Encode Parameters

Low-latency streaming configures the encoder with:

1. **Zero B-frames**: B-frames require buffering future frames before encoding, adding latency equal to the B-frame depth. For live streaming, `consecutiveBFrameCount = 0` and `predictionMode = SINGLE_REFERENCE_KHR` (AV1) must be used.

2. **CBR with small VBV buffer**: A small `virtualBufferSizeInMs` (e.g., 50–200 ms) forces the encoder to smooth bitrate over a short window, reducing buffering at the network layer. The tradeoff is that the encoder cannot save bits from easy frames for complex frames across a longer window.

3. **Forced IDR on scene cut**: When a scene cut is detected (by the application, using a GPU compute shader computing histogram differences between frames), `IdrPicFlag = 1` is set for the next frame. This prevents the encoder from using a distant reference that shares no content with the current scene, which would otherwise cause quality spikes.

4. **Low-latency flag**: `VkVideoEncodeRateControlInfoKHR` includes support for low-latency flags (`VK_VIDEO_ENCODE_RATE_CONTROL_FLAG_DEFAULT_KHR`) when combined with CBR and zero B-frames; individual driver implementations may expose additional hints via `VkVideoEncodeH264RateControlInfoKHR.flags`.

```c
/* src: low-latency streaming encode configuration */
VkVideoEncodeH264RateControlInfoKHR h264_rc_ll = {
    .sType                  = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_RATE_CONTROL_INFO_KHR,
    .flags                  = VK_VIDEO_ENCODE_H264_RATE_CONTROL_REFERENCE_PATTERN_FLAT_BIT_KHR,
    .gopFrameCount          = 0,    /* infinite GOP, IDRs are application-triggered */
    .idrPeriod              = 0,    /* no periodic IDR */
    .consecutiveBFrameCount = 0,    /* no B-frames */
    .temporalLayerCount     = 1,
};
VkVideoEncodeRateControlInfoKHR rc_ll = {
    .sType           = VK_STRUCTURE_TYPE_VIDEO_ENCODE_RATE_CONTROL_INFO_KHR,
    .pNext           = &h264_rc_ll,
    .rateControlMode = VK_VIDEO_ENCODE_RATE_CONTROL_MODE_CBR_BIT_KHR,
    .layerCount      = 1,
    .pLayers         = &rc_layer_4mbps,
    .virtualBufferSizeInMs         = 100,   /* 100ms VBV buffer */
    .initialVirtualBufferSizeInMs  = 0,
};
```

### OBS Studio Vulkan Capture and Encode

OBS Studio (Open Broadcaster Software) uses a Vulkan capture layer to intercept game frames rendered via Vulkan. The OBS Vulkan layer (covered in detail in Ch153) inserts itself into the VkQueue submit path via the Vulkan layer mechanism, capturing the swapchain image before presentation.

With Vulkan Video encode, the captured `VkImage` can be submitted directly to `vkCmdEncodeVideoKHR` without leaving the GPU:

```
Application (game)
  → Vulkan draw calls → swapchain VkImage (rendered frame)
  → OBS Vulkan capture layer intercepts vkQueuePresentKHR
  → Captured VkImage layout transition: PRESENT_SRC → VIDEO_ENCODE_SRC
  → vkCmdEncodeVideoKHR (h264_vulkan or hevc_vulkan)
  → Encoded NAL units → VkBuffer → CPU read → RTMP/SRT stream
```

The layout transition is a single `VkImageMemoryBarrier2` from `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR` to `VK_IMAGE_LAYOUT_VIDEO_ENCODE_SRC_KHR`, a zero-copy path that avoids the screenshot download OBS historically performed on CPU. This Vulkan-native capture path is under active development in OBS as of 2025. (See Ch153 for the OBS Vulkan pipeline architecture.)

### ALVR and WiVRn Wireless VR Streaming

ALVR (Air Light VR) and WiVRn are open-source solutions for streaming PC VR content wirelessly to standalone headsets (Meta Quest, etc.) over Wi-Fi. Both intercept the SteamVR/OpenXR rendered eye textures and encode them for transmission.

On Linux, ALVR uses a Vulkan WSI layer to intercept `vkQueuePresentKHR` calls from `vrcompositor`, capturing the stereo render target `VkImage`. Newer kernel and Mesa versions prefer the Vulkan encode path over VA-API for capture-to-encode because:

- The rendered stereo image is already a `VkImage` — no DMA-BUF export or VA-API surface import is required.
- The Vulkan Video encode session lives on the same `VkDevice` as the VR renderer, enabling a single GPU command buffer that contains both the final stereo blit and the encode dispatch.
- Timeline semaphores synchronise encode completion with network transmission without CPU involvement.

[Source: How ALVR works — ALVR Wiki](https://github.com/alvr-org/ALVR/wiki/How-ALVR-works)

The low-latency encode configuration for wireless VR is the most demanding: the total encode-network-decode-display budget is typically 20–40 ms to avoid motion sickness. This requires:

- **H.264 or H.265** (not AV1, which may have higher decode latency on the headset SoC)
- **Zero B-frames, zero lookahead**
- **CBR at maximum bitrate** the Wi-Fi channel supports (~100–200 Mbit/s for Wi-Fi 6)
- **Forced IDR every 1–2 seconds** to limit error propagation over lossy Wi-Fi

WiVRn similarly uses VA-API or Vulkan encode depending on what is available. Where both are available, the application layer chooses based on which has lower encode submission latency on the target hardware.

---

## 10. Integrations

**Chapter 50 — Vulkan Video Decode**: This chapter (Ch165) is the encode counterpart to Ch50. Ch50 covers `VK_KHR_video_queue` infrastructure, decode lifecycle, H.264/H.265/AV1 decode, DPB management, and FFmpeg decode hwaccel. The two chapters share the queue discovery and session lifecycle model described in Ch50 but diverge completely at the operation layer (`VK_KHR_video_decode_queue` vs `VK_KHR_video_encode_queue`). Zero-copy transcode pipelines combine both chapters' content: decode from Ch50, an image layout transition barrier, and encode from this chapter.

**Chapter 26 — VA-API Encode**: VA-API is the traditional hardware encode API on Linux, providing `VAEntrypointEncSlice` (H.264/H.265) and `VAEntrypointEncPicture` (JPEG) entry points via `vaCreateConfig` and `vaRenderPicture`. Vulkan Video encode (`VK_KHR_video_encode_queue`) is the Vulkan-native alternative to VA-API encode, offering the advantage of pipeline integration (no DMA-BUF surface export for cross-API sharing). Ch26 covers the VA-API encode path in detail; this chapter covers the Vulkan path.

**Chapter 57 — FFmpeg**: FFmpeg is the primary consumer of Vulkan Video encode through the `h264_vulkan`, `hevc_vulkan`, and `av1_vulkan` encoder names. Ch57 covers FFmpeg's codec architecture, filter graph, and hardware context framework in depth. Section 7 of this chapter shows how that framework is applied to Vulkan encode.

**Chapter 58 — GStreamer Encode Pipeline**: GStreamer's `vulkah264enc`, `vulkanh265enc`, and `vulkav1enc` elements (Section 8) build on the `GstVulkanEncoder` base class. Ch58 covers the broader GStreamer architecture, the `GstVideoEncoder` base class hierarchy, and the hardware acceleration abstraction that makes Vulkan Video elements interchangeable with VA-API-backed elements.

**Chapter 79 — Remote Display and Streaming**: Ch79 covers PipeWire screen sharing, WebRTC, and RTSP streaming pipelines. The low-latency Vulkan Video encode described in Section 9 feeds directly into these streaming stacks. WiVRn and ALVR (Section 9) are examples of the wireless VR streaming systems covered in Ch79's remote display section.

**Chapter 153 — OBS Studio GPU Pipeline**: Ch153 covers OBS Studio's Vulkan capture layer, game capture mechanism, and the overall GPU pipeline from rendered frame to encoded stream. Section 9 of this chapter describes how the captured `VkImage` flows from OBS's Vulkan layer into `vkCmdEncodeVideoKHR` for zero-copy GPU encode.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
