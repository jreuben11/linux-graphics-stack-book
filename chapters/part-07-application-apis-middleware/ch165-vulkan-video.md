# Chapter 165: Vulkan Video: Encode and Decode Extensions

**Target audiences**: Application developers integrating hardware video decode or encode into Vulkan pipelines; driver developers implementing `VK_KHR_video_decode_h264` and related extensions; and engineers building zero-copy transcoding pipelines.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Vulkan Video Extension Family](#vulkan-video-extension-family)
3. [Video Queues and Capabilities](#video-queues-and-capabilities)
4. [Video Session and Parameters](#video-session-and-parameters)
5. [H.264 Decode Pipeline](#h264-decode-pipeline)
6. [H.265 and AV1 Decode](#h265-and-av1-decode)
7. [H.264 and H.265 Encode](#h264-and-h265-encode)
8. [Zero-Copy: Vulkan Video with Compute and Graphics](#zero-copy-vulkan-video-with-compute-and-graphics)
9. [Mesa RADV and ANV Implementation](#mesa-radv-and-anv-implementation)
10. [Integrations](#integrations)

---

## Introduction

`VK_KHR_video_queue` and its codec-specific companions (`VK_KHR_video_decode_h264`, `VK_KHR_video_decode_h265`, `VK_KHR_video_decode_av1`, `VK_KHR_video_encode_h264`, `VK_KHR_video_encode_h265`) allow applications to perform hardware video decode and encode entirely within Vulkan, without touching VA-API, NVENC, or any other vendor API.

The key advantage over VA-API is **pipeline integration**: a decoded frame stays in a Vulkan image and can be consumed immediately by compute or graphics shaders — no cross-API buffer transfer, no copy through VA-API DRM Prime.

Vulkan Video reached provisional status in 2022 and became final with Vulkan 1.3 extension revisions in 2023. Mesa RADV and ANV both implement H.264/H.265 decode.

Sources: [Vulkan Video spec](https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap50.html) | [Mesa video decode](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/vulkan/video)

---

## Vulkan Video Extension Family

### Extension Hierarchy

```
VK_KHR_video_queue              ← base: queue, session, parameters
  ├── VK_KHR_video_decode_queue ← decode command buffer
  │   ├── VK_KHR_video_decode_h264
  │   ├── VK_KHR_video_decode_h265
  │   └── VK_KHR_video_decode_av1
  └── VK_KHR_video_encode_queue ← encode command buffer
      ├── VK_KHR_video_encode_h264
      └── VK_KHR_video_encode_h265
```

### Feature Flags

```c
/* Check video decode support: */
VkPhysicalDeviceVideoDecodeH264FeaturesKHR h264_features = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VIDEO_DECODE_H264_FEATURES_KHR,
};
VkPhysicalDeviceFeatures2 features2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2,
    .pNext = &h264_features,
};
vkGetPhysicalDeviceFeatures2(phys_dev, &features2);
/* h264_features.videoDecodeH264 must be VK_TRUE */
```

---

## Video Queues and Capabilities

### Video Queue Family

Video decode/encode use dedicated queue families (not graphics or compute):

```c
/* Find a video decode queue: */
uint32_t queue_family_count;
vkGetPhysicalDeviceQueueFamilyProperties2(phys_dev, &queue_family_count, NULL);

VkQueueFamilyVideoPropertiesKHR *video_props =
    calloc(queue_family_count, sizeof(*video_props));
VkQueueFamilyProperties2 *qf_props =
    calloc(queue_family_count, sizeof(*qf_props));

for (uint32_t i = 0; i < queue_family_count; i++) {
    qf_props[i].sType = VK_STRUCTURE_TYPE_QUEUE_FAMILY_PROPERTIES_2;
    qf_props[i].pNext = &video_props[i];
    video_props[i].sType = VK_STRUCTURE_TYPE_QUEUE_FAMILY_VIDEO_PROPERTIES_KHR;
}
vkGetPhysicalDeviceQueueFamilyProperties2(phys_dev, &queue_family_count, qf_props);

uint32_t video_decode_qf = UINT32_MAX;
for (uint32_t i = 0; i < queue_family_count; i++) {
    if (qf_props[i].queueFamilyProperties.queueFlags & VK_QUEUE_VIDEO_DECODE_BIT_KHR) {
        if (video_props[i].videoCodecOperations &
            VK_VIDEO_CODEC_OPERATION_DECODE_H264_BIT_KHR)
        {
            video_decode_qf = i;
            break;
        }
    }
}
```

### Video Capabilities

```c
/* Query H.264 decode capabilities: */
VkVideoDecodeH264CapabilitiesKHR h264_caps = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_DECODE_H264_CAPABILITIES_KHR,
};
VkVideoDecodeCapabilitiesKHR decode_caps = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_DECODE_CAPABILITIES_KHR,
    .pNext = &h264_caps,
};
VkVideoCapabilitiesKHR caps = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_CAPABILITIES_KHR,
    .pNext = &decode_caps,
};

VkVideoProfileInfoKHR profile = {
    .sType               = VK_STRUCTURE_TYPE_VIDEO_PROFILE_INFO_KHR,
    .videoCodecOperation = VK_VIDEO_CODEC_OPERATION_DECODE_H264_BIT_KHR,
    .chromaSubsampling   = VK_VIDEO_CHROMA_SUBSAMPLING_420_BIT_KHR,
    .lumaBitDepth        = VK_VIDEO_COMPONENT_BIT_DEPTH_8_BIT_KHR,
    .chromaBitDepth      = VK_VIDEO_COMPONENT_BIT_DEPTH_8_BIT_KHR,
};
vkGetPhysicalDeviceVideoCapabilitiesKHR(phys_dev, &profile, &caps);
/* caps.minCodedExtent, caps.maxCodedExtent: supported resolution range */
/* h264_caps.maxLevelIdc: highest H.264 level supported */
```

---

## Video Session and Parameters

### VkVideoSessionKHR

The video session holds codec state (reference picture manager, DPB):

```c
VkVideoSessionCreateInfoKHR session_ci = {
    .sType              = VK_STRUCTURE_TYPE_VIDEO_SESSION_CREATE_INFO_KHR,
    .queueFamilyIndex   = video_decode_qf,
    .pVideoProfile      = &profile,
    .pictureFormat      = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,  /* NV12 */
    .maxCodedExtent     = { 1920, 1080 },
    .referencePictureFormat = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,
    .maxDpbSlots        = 17,   /* H.264 max DPB size */
    .maxActiveReferencePictures = 16,
    .pStdHeaderVersion  = &std_header_version,
};
VkVideoSessionKHR session;
vkCreateVideoSessionKHR(device, &session_ci, NULL, &session);

/* Allocate session memory: */
uint32_t mem_req_count;
vkGetVideoSessionMemoryRequirementsKHR(device, session, &mem_req_count, NULL);
VkVideoSessionMemoryRequirementsKHR *mem_reqs =
    malloc(mem_req_count * sizeof(*mem_reqs));
vkGetVideoSessionMemoryRequirementsKHR(device, session, &mem_req_count, mem_reqs);
/* ... allocate and bind memory for each requirement ... */
```

### VkVideoSessionParametersKHR

For H.264, the session parameters hold SPS/PPS (Sequence/Picture Parameter Sets):

```c
StdVideoH264SequenceParameterSet sps = {
    /* parsed from bitstream: */
    .profile_idc     = STD_VIDEO_H264_PROFILE_IDC_HIGH,
    .level_idc       = STD_VIDEO_H264_LEVEL_IDC_4_1,
    .chroma_format_idc = STD_VIDEO_H264_CHROMA_FORMAT_IDC_420,
    .pic_width_in_mbs_minus1    = (1920 / 16) - 1,
    .pic_height_in_map_units_minus1 = (1088 / 16) - 1,
    /* ... */
};

VkVideoDecodeH264SessionParametersAddInfoKHR h264_params_add = {
    .sType       = VK_STRUCTURE_TYPE_VIDEO_DECODE_H264_SESSION_PARAMETERS_ADD_INFO_KHR,
    .stdSPSCount = 1,
    .pStdSPSs    = &sps,
    .stdPPSCount = 1,
    .pStdPPSs    = &pps,
};
VkVideoSessionParametersCreateInfoKHR params_ci = {
    .sType                    = VK_STRUCTURE_TYPE_VIDEO_SESSION_PARAMETERS_CREATE_INFO_KHR,
    .videoSession             = session,
    .pNext                    = &h264_params_add,
};
VkVideoSessionParametersKHR params;
vkCreateVideoSessionParametersKHR(device, &params_ci, NULL, &params);
```

---

## H.264 Decode Pipeline

### DPB (Decoded Picture Buffer)

H.264 decoding requires a DPB for reference frames. Each DPB slot is a Vulkan image:

```c
/* Create DPB images (reference frame storage): */
VkImageCreateInfo dpb_img_ci = {
    .sType       = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .imageType   = VK_IMAGE_TYPE_2D,
    .format      = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,  /* NV12 */
    .extent      = { 1920, 1088, 1 },
    .mipLevels   = 1,
    .arrayLayers = 1,
    .usage       = VK_IMAGE_USAGE_VIDEO_DECODE_DPB_BIT_KHR,
};
/* Also need the output image: */
VkImageCreateInfo out_img_ci = dpb_img_ci;
out_img_ci.usage = VK_IMAGE_USAGE_VIDEO_DECODE_DST_BIT_KHR
                 | VK_IMAGE_USAGE_SAMPLED_BIT;  /* for compute/graphics */
```

### Decode Command Recording

```c
/* Begin video coding scope: */
VkVideoBeginCodingInfoKHR begin_ci = {
    .sType          = VK_STRUCTURE_TYPE_VIDEO_BEGIN_CODING_INFO_KHR,
    .videoSession   = session,
    .videoSessionParameters = params,
    .referenceSlotCount = dpb_slot_count,
    .pReferenceSlots    = dpb_slots,
};
vkCmdBeginVideoCodingKHR(cmd, &begin_ci);

/* H.264-specific decode info: */
VkVideoDecodeH264PictureInfoKHR h264_pic = {
    .sType         = VK_STRUCTURE_TYPE_VIDEO_DECODE_H264_PICTURE_INFO_KHR,
    .pStdPictureInfo = &slice_header,
    .sliceCount    = 1,
    .pSliceOffsets = &slice_offset,
};
VkVideoDecodeInfoKHR decode_info = {
    .sType          = VK_STRUCTURE_TYPE_VIDEO_DECODE_INFO_KHR,
    .pNext          = &h264_pic,
    .srcBuffer      = bitstream_buf,     /* compressed H.264 bitstream */
    .srcBufferOffset = 0,
    .srcBufferRange = bitstream_size,
    .dstPictureResource = {
        .codedOffset = {0, 0},
        .codedExtent = {1920, 1080},
        .baseArrayLayer = 0,
        .imageViewBinding = output_image_view,
    },
    .pSetupReferenceSlot = &setup_slot,  /* slot for this frame if it's a ref */
    .referenceSlotCount  = ref_count,
    .pReferenceSlots     = ref_slots,
};
vkCmdDecodeVideoKHR(cmd, &decode_info);

/* End scope: */
VkVideoEndCodingInfoKHR end_ci = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_END_CODING_INFO_KHR };
vkCmdEndVideoCodingKHR(cmd, &end_ci);
```

---

## H.265 and AV1 Decode

### H.265 (HEVC) Decode

`VK_KHR_video_decode_h265` follows the same pattern with H.265-specific parameter sets (VPS, SPS, PPS):

```c
VkVideoDecodeH265PictureInfoKHR h265_pic = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_DECODE_H265_PICTURE_INFO_KHR,
    .pStdPictureInfo = &hevc_slice_header,
    .sliceSegmentCount = slice_count,
    .pSliceSegmentOffsets = offsets,
};
```

H.265 supports tiles and slices; `pSliceSegmentOffsets` lists the offset of each slice segment NAL unit within the bitstream buffer.

### AV1 Decode

`VK_KHR_video_decode_av1` (finalised 2024) supports AV1's tile-based bitstream:

```c
VkVideoDecodeAV1PictureInfoKHR av1_pic = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_DECODE_AV1_PICTURE_INFO_KHR,
    .pStdPictureInfo  = &av1_frame_header,
    .tileCount        = frame_tile_count,
    .pTileOffsets     = tile_offsets,
    .pTileSizes       = tile_sizes,
};
```

---

## H.264 and H.265 Encode

### Encode Session

```c
VkVideoProfileInfoKHR encode_profile = {
    .sType               = VK_STRUCTURE_TYPE_VIDEO_PROFILE_INFO_KHR,
    .videoCodecOperation = VK_VIDEO_CODEC_OPERATION_ENCODE_H264_BIT_KHR,
    .chromaSubsampling   = VK_VIDEO_CHROMA_SUBSAMPLING_420_BIT_KHR,
    .lumaBitDepth        = VK_VIDEO_COMPONENT_BIT_DEPTH_8_BIT_KHR,
    .chromaBitDepth      = VK_VIDEO_COMPONENT_BIT_DEPTH_8_BIT_KHR,
};

VkVideoSessionCreateInfoKHR encode_session_ci = {
    /* ... */
    .pVideoProfile  = &encode_profile,
    .pictureFormat  = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,
};
```

### Encode Command

```c
VkVideoEncodeH264PictureInfoKHR h264_enc_pic = {
    .sType = VK_STRUCTURE_TYPE_VIDEO_ENCODE_H264_PICTURE_INFO_KHR,
    .naluSliceEntryCount = 1,
    .pNaluSliceEntries   = &slice_entry,
    .pStdPictureInfo     = &std_pic_info,
};
VkVideoEncodeInfoKHR encode_info = {
    .sType            = VK_STRUCTURE_TYPE_VIDEO_ENCODE_INFO_KHR,
    .pNext            = &h264_enc_pic,
    .srcPictureResource = {
        .imageViewBinding = source_image_view,   /* NV12 source frame */
        .codedExtent      = { 1920, 1080 },
    },
    .dstBuffer        = output_bitstream_buf,
    .dstBufferOffset  = 0,
    .dstBufferRange   = output_bitstream_size,
};
vkCmdEncodeVideoKHR(cmd, &encode_info);
```

---

## Zero-Copy: Vulkan Video with Compute and Graphics

### The Zero-Copy Pipeline

```
H.264 bitstream (VkBuffer)
  → vkCmdDecodeVideoKHR → decoded NV12 (VkImage)
  → Pipeline barrier (VIDEO_DECODE → COMPUTE)
  → vkCmdDispatch (colour space convert NV12→RGB, or ML inference)
  → vkCmdDraw (render to screen directly)
```

No CPU involvement, no DRM Prime export, no VA-API.

### Ownership Transfer

Images transition ownership between the video queue and graphics/compute:

```c
/* Transfer image from video decode queue to compute queue: */
VkImageMemoryBarrier2 barrier = {
    .sType         = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_VIDEO_DECODE_BIT_KHR,
    .srcAccessMask = VK_ACCESS_2_VIDEO_DECODE_WRITE_BIT_KHR,
    .dstStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_READ_BIT,
    .oldLayout     = VK_IMAGE_LAYOUT_VIDEO_DECODE_DST_KHR,
    .newLayout     = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL,
    .srcQueueFamilyIndex = video_decode_qf,
    .dstQueueFamilyIndex = compute_qf,
    .image         = decoded_image,
};
```

---

## Mesa RADV and ANV Implementation

### RADV Video Decode

RADV implements H.264/H.265/AV1 decode using the VCN (Video Core Next) block on RDNA2/3:

```bash
# Check Vulkan video support on AMD:
vulkaninfo | grep -i "video\|VK_KHR_video"
# Should show: VK_KHR_video_decode_h264, VK_KHR_video_decode_h265, VK_KHR_video_decode_av1
```

```c
/* radv/video/radv_video.c */
void radv_CmdDecodeVideoKHR(VkCommandBuffer commandBuffer,
    const VkVideoDecodeInfoKHR *pDecodeInfo)
{
    /* Program VCN register MMSCH with decode job: */
    radv_emit_vcn_decode(cmd_buffer, pDecodeInfo);
    /* VCN runs asynchronously; signals timeline semaphore on completion */
}
```

VCN (Video Core Next) is AMD's dedicated video accelerator block, separate from the shader engines. VCN4 (RDNA3) supports AV1 decode.

### ANV Video Decode

ANV implements video decode using Intel's VDBOX (Video Decode Box) engine:

```bash
# Check on Intel:
vulkaninfo | grep "VK_KHR_video"
```

```c
/* anv/video/anv_video.c */
void anv_CmdDecodeVideoKHR(VkCommandBuffer commandBuffer,
    const VkVideoDecodeInfoKHR *pDecodeInfo)
{
    /* Emit MFX/VDENC command sequence for H.264 decode via VDBOX: */
    anv_batch_emit(&cmd_buffer->batch, GENX(MFD_AVC_DPB_STATE), dpb);
    anv_batch_emit(&cmd_buffer->batch, GENX(MFD_AVC_SLICEADDR), slice);
    anv_batch_emit(&cmd_buffer->batch, GENX(MFD_AVC_BSD_OBJECT), bsd_obj);
}
```

---

## Integrations

- **Ch19 (Vulkan Architecture)** — video queues use the same VkCommandBuffer / VkQueue model as graphics and compute; synchronisation uses the same VkSemaphore and VkFence primitives
- **Ch20 (SPIR-V)** — shaders that process decoded video frames (NV12→RGB colour conversion, SSIM metric, ML inference) are standard SPIR-V compute shaders
- **Ch26 (Hardware Video)** — Vulkan Video is an alternative to VA-API for hardware decode/encode; both ultimately drive the same VCN/VDBOX hardware blocks
- **Ch22 (RADV)** — RADV's Vulkan Video implementation uses AMD VCN hardware; shares the same GEM BO management as 3D rendering
- **Ch23 (ANV)** — ANV uses Intel VDBOX for video decode; Xe2 adds hardware AV1 encode
- **Ch150 (EGL/DMA-BUF)** — video decoded via VA-API can be imported into Vulkan via DMA-BUF + EGLImage; Vulkan Video avoids this cross-API step entirely
