# Chapter 164: GPU Power Management: Runtime PM, DVFS, and Power Caps

**Target audiences**: Driver developers implementing GPU power management; systems engineers optimising power consumption on laptops and servers; and embedded developers managing thermal and power budgets on mobile SoCs.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Linux Runtime PM Framework](#linux-runtime-pm-framework)
3. [GPU DVFS: Dynamic Voltage and Frequency Scaling](#gpu-dvfs-dynamic-voltage-and-frequency-scaling)
4. [AMD GPU Power Management: dpm and Overdrive](#amd-gpu-power-management-dpm-and-overdrive)
5. [Intel GPU Power Management: RC6 and Turbo](#intel-gpu-power-management-rc6-and-turbo)
6. [NVIDIA Power Management: Turing and Ampere](#nvidia-power-management-turing-and-ampere)
7. [Power Capping and TDP Control](#power-capping-and-tdp-control)
8. [Thermal Management](#thermal-management)
9. [Power Profiling Tools](#power-profiling-tools)
10. [Integrations](#integrations)

---

## Introduction

GPU power management is critical at both ends of the performance spectrum: laptops need GPUs to sleep aggressively to extend battery life, while data-centre GPUs need precise power caps to stay within rack power budgets. The Linux kernel provides both via the runtime PM framework (for sleep/wake) and the DVFS infrastructure (for frequency/voltage scaling).

GPU power state has a direct impact on application latency: waking from deep sleep can add 1–30 ms of first-frame latency. Power management tuning requires balancing idle power savings against wake-up latency.

---

## Linux Runtime PM Framework

### Runtime PM Concepts

Linux runtime PM (`include/linux/pm_runtime.h`) manages device power states at runtime:

```
D0 (active) → D0i2 (idle, partial clock gate) → D3 (off, ~0 mW)
              ↑ resume (1–30ms)  ↓ suspend (delay: autosuspend_delay_ms)
```

```c
/* GPU driver (e.g. amdgpu_drv.c): enable autosuspend: */
pm_runtime_set_autosuspend_delay(&pdev->dev, 2000);  /* 2 seconds */
pm_runtime_use_autosuspend(&pdev->dev);
pm_runtime_put_autosuspend(&pdev->dev);  /* allow suspend after 2s idle */
```

### GPU Runtime Suspend

When the GPU has no active workload for `autosuspend_delay_ms`, the driver suspends it:

```c
/* amdgpu/amdgpu_drv.c: runtime suspend callback */
static int amdgpu_pmops_runtime_suspend(struct device *dev)
{
    struct drm_device *drm_dev = dev_get_drvdata(dev);
    struct amdgpu_device *adev = drm_dev->dev_private;

    /* Stop all engines: */
    amdgpu_device_prepare(drm_dev);
    amdgpu_device_suspend(drm_dev, false);
    /* Gate PCI link: */
    pci_save_state(pdev);
    pci_set_power_state(pdev, PCI_D3cold);
    return 0;
}
```

### Checking Runtime PM State

```bash
# Check GPU runtime PM status:
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
# Values: active, suspended, suspending, resuming

# Force runtime suspend (for testing):
echo on > /sys/bus/pci/devices/0000:01:00.0/power/control
echo auto > /sys/bus/pci/devices/0000:01:00.0/power/control

# Check suspend/resume count:
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_suspended_time
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_active_time
```

---

## GPU DVFS: Dynamic Voltage and Frequency Scaling

### devfreq Framework

Linux's `devfreq` framework (`drivers/devfreq/`) manages GPU frequency scaling:

```bash
# GPU devfreq (typical on ARM/mobile):
ls /sys/class/devfreq/
# e.g. /sys/class/devfreq/1c40000.gpu/

cat /sys/class/devfreq/*/cur_freq        # current frequency
cat /sys/class/devfreq/*/available_frequencies  # all supported freqs
cat /sys/class/devfreq/*/governor        # current governor (simple_ondemand, etc.)
echo performance > /sys/class/devfreq/*/governor  # pin to max frequency
```

### devfreq Governors

| Governor | Description | Use Case |
|---|---|---|
| `simple_ondemand` | Scale up on load, scale down after threshold | Default; balances perf/power |
| `performance` | Pin to maximum frequency | Benchmarking |
| `powersave` | Pin to minimum frequency | Maximum battery life |
| `userspace` | Manual frequency control | Profiling, testing |
| `passive` | Follows another device's policy | Mobile SoC (CPU-linked) |

```c
/* Registering a GPU with devfreq (panfrost example): */
/* panfrost/panfrost_devfreq.c */
static int panfrost_devfreq_target(struct device *dev,
    unsigned long *target_freq, u32 flags)
{
    struct panfrost_device *pfdev = dev_get_drvdata(dev);
    struct dev_pm_opp *opp = dev_pm_opp_find_freq_ceil(dev, target_freq);
    unsigned long voltage = dev_pm_opp_get_voltage(opp);

    /* Set voltage first when scaling up, frequency then: */
    if (*target_freq > pfdev->current_freq) {
        regulator_set_voltage(pfdev->regulator, voltage, voltage);
        clk_set_rate(pfdev->core_clk, *target_freq);
    } else {
        clk_set_rate(pfdev->core_clk, *target_freq);
        regulator_set_voltage(pfdev->regulator, voltage, voltage);
    }
    pfdev->current_freq = *target_freq;
    return 0;
}
```

---

## AMD GPU Power Management: dpm and Overdrive

### DPM: Dynamic Power Management

AMD's DPM (Dynamic Power Management) manages GPU power states (P-states) and frequencies:

```bash
# AMD GPU power profile:
cat /sys/class/drm/card0/device/power_dpm_state
# Values: battery, balanced, performance

echo performance > /sys/class/drm/card0/device/power_dpm_state

# Power profile modes (RDNA3):
cat /sys/class/drm/card0/device/pp_power_profile_mode
# Lists: BOOTUP_DEFAULT, 3D_FULL_SCREEN, POWER_SAVING, VIDEO, VR, COMPUTE, CUSTOM

echo 1 > /sys/class/drm/card0/device/pp_power_profile_mode  # 3D Full Screen

# Current P-state and clocks:
cat /sys/kernel/debug/dri/0/amdgpu_pm_info
```

### AMDGPU DPM Internals

```c
/* amdgpu/amdgpu_dpm.c */
struct amdgpu_dpm_policy {
    enum amd_dpm_forced_level level;
    /* auto: driver selects p-state based on load */
    /* low: minimum performance, maximum power saving */
    /* high: maximum performance */
    /* manual: user controls individual clock/voltage levels */
};

int amdgpu_dpm_set_performance_level(struct amdgpu_device *adev,
    enum amd_dpm_forced_level level)
{
    if (adev->pm.funcs->set_performance_level)
        return adev->pm.funcs->set_performance_level(adev, level);
    return -EINVAL;
}
```

### SMU (System Management Unit)

RDNA2/RDNA3 GPUs use the SMU (System Management Unit) — an embedded ARM Cortex-M4 microcontroller — for power management:

```c
/* amdgpu/smu_v13_0.c: communicate with SMU via mailbox: */
static int smu_cmn_send_msg_without_waiting(struct smu_context *smu,
    uint16_t msg, uint32_t param)
{
    /* Write command to SMU mailbox register: */
    WREG32_SOC15(MP1, 0, mmMP1_SMN_C2PMSG_66, param);
    WREG32_SOC15(MP1, 0, mmMP1_SMN_C2PMSG_82, msg);
    return 0;
}
```

### AMD Overdrive

Overdrive allows manual GPU and memory frequency/voltage control beyond default limits:

```bash
# Enable Overdrive (requires kernel boot param: amdgpu.ppfeaturemask=0xffffffff)
# or:
echo "oc" > /sys/class/drm/card0/device/power_dpm_force_performance_level

# Adjust GPU clock via pp_od_clk_voltage:
echo "s 1 1800 1000" > /sys/class/drm/card0/device/pp_od_clk_voltage  # state 1: 1800MHz, 1.0V
echo "c" > /sys/class/drm/card0/device/pp_od_clk_voltage  # commit
```

---

## Intel GPU Power Management: RC6 and Turbo

### RC6: Render Standby

Intel's RC6 (Render Clock-Gating) powers down the GPU render engine during idle periods:

```bash
# Check RC6 state:
cat /sys/class/drm/card0/gt/gt0/rc6_enable
# 1 = enabled (default on laptops)

# RC6 residency (time in deep sleep):
cat /sys/class/drm/card0/gt/gt0/rc6_residency_ms

# RC6+ (deeper sleep):
cat /sys/class/drm/card0/gt/gt0/rc6p_residency_ms
```

RC6 on modern Intel gives near-zero GPU power consumption during desktop idle — typically 100–500 mW.

### GT Boost / GuC

Intel Gen11+ uses **GuC** (Graphics micro-Controller) for power management:

```c
/* drivers/gpu/drm/i915/gt/uc/intel_guc_pm.c */
static int guc_enable_gt_pm(struct intel_guc *guc)
{
    /* GuC takes over GT power management: */
    return intel_guc_send(guc, INTEL_GUC_ACTION_SETUP_PC_GUCRC, 0);
}
```

GuC RC enables tighter residency tracking and faster wake-up than software RC6.

### Intel Turbo

Intel GPU Turbo (analogous to CPU Turbo Boost) allows brief frequency bursts above TDP:

```bash
# GPU frequency range:
cat /sys/class/drm/card0/gt/gt0/rps_min_freq_mhz  # minimum
cat /sys/class/drm/card0/gt/gt0/rps_max_freq_mhz  # maximum
cat /sys/class/drm/card0/gt/gt0/rps_cur_freq_mhz  # current
cat /sys/class/drm/card0/gt/gt0/rps_boost_freq_mhz # boost

# Force specific frequency (for benchmarking):
echo 1200 > /sys/class/drm/card0/gt/gt0/rps_min_freq_mhz
echo 1200 > /sys/class/drm/card0/gt/gt0/rps_max_freq_mhz
```

---

## NVIDIA Power Management: Turing and Ampere

### Open Kernel Module PM

NVIDIA's open kernel modules (`nvidia-open`, kernel 5.18+) expose limited power management:

```bash
# NVIDIA power state:
nvidia-smi -q -d POWER | head -20
# Check dynamic boost:
nvidia-smi --query-gpu=power.draw,power.limit --format=csv

# Runtime PM (requires kernel param: NVreg_DynamicPowerManagement=0x02):
cat /proc/driver/nvidia/params | grep DynamicPower
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
```

### NVIDIA Power Limits

```bash
# Check TDP limits:
nvidia-smi -q | grep "Default Power"
# Set power limit (watt):
sudo nvidia-smi --power-limit=150    # cap to 150W (for thermal/power budget)
sudo nvidia-smi --power-limit=350    # restore default
```

### MIG (Multi-Instance GPU) Power

On A100/H100, MIG partitions the GPU and assigns power budgets per instance:

```bash
nvidia-smi mig -lgip   # list GPU instance profiles
nvidia-smi mig -cgi 9,9  # create 2 MIG instances (each gets ~half power)
```

---

## Power Capping and TDP Control

### RAPL for GPU (Intel Platforms)

Intel RAPL (Running Average Power Limit) exposes GPU TDP on some platforms:

```bash
# GPU TDP via RAPL (PowerLimit):
cat /sys/class/powercap/intel-rapl/intel-rapl:0/name
# Domains: package, dram, uncore, core
# Uncore = GPU on integrated platforms

# Read GPU power (via `turbostat`):
turbostat --quiet --show GFXWatt sleep 1
```

### hwmon Power Monitoring

```bash
# AMD GPU power via hwmon:
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_average  # µW

# Intel:
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_average

# All GPU sensors:
sensors | grep -A5 "amdgpu\|nouveau\|radeon"
```

### cgroups v2 and GPU Power

GPU power budget assignment via cgroups is not directly supported in mainline Linux (no GPU bandwidth controller). However:
- CPU cgroups affect GPU indirectly (less CPU = fewer GPU submissions)
- NVIDIA MIG provides hardware-enforced GPU partitioning
- AMD compute isolation uses KFD contexts with priority classes

---

## Thermal Management

### Thermal Zones and Trips

```bash
# GPU thermal zone:
grep -r "gpu\|amdgpu\|nouveau" /sys/class/thermal/thermal_zone*/type
# e.g. /sys/class/thermal/thermal_zone4/type: amdgpu

# Current GPU temperature:
cat /sys/class/thermal/thermal_zone4/temp    # millidegrees Celsius
# e.g. 42000 = 42°C

# Trip points (throttle/shutdown temperatures):
cat /sys/class/thermal/thermal_zone4/trip_point_0_temp
cat /sys/class/thermal/thermal_zone4/trip_point_0_type  # passive (throttle)
cat /sys/class/thermal/thermal_zone4/trip_point_1_temp
cat /sys/class/thermal/thermal_zone4/trip_point_1_type  # critical (shutdown)
```

### AMD Thermal Throttling

AMD GPUs implement hardware thermal throttling via the SMU:

```bash
# Check if GPU is throttling:
cat /sys/kernel/debug/dri/0/amdgpu_pm_info | grep -i "throttl\|limit"
# Or via hwmon:
cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_emergency  # emergency trip
```

```c
/* amdgpu: thermal throttle notification from SMU: */
static void amdgpu_thermal_work_handler(struct work_struct *work)
{
    struct amdgpu_device *adev = container_of(work,
        struct amdgpu_device, thermal_work);
    /* Log thermal event: */
    dev_warn(adev->dev, "GPU over temperature: %d°C, throttling",
             adev->pm.dpm.thermal.max_temp / 1000);
    amdgpu_dpm_set_performance_level(adev, AMD_DPM_FORCED_LEVEL_LOW);
}
```

---

## Power Profiling Tools

### powertop

```bash
# GPU wakeup analysis:
sudo powertop
# Shows GPU wakeup rate; high rate means poor PM

# GPU power estimate:
sudo powertop --auto-tune  # enable all power saving tunables
```

### amdgpu_top / nvtop

```bash
# AMD real-time GPU monitor (power, freq, utilisation):
amdgpu_top          # https://github.com/Umio-Yasuno/amdgpu_top
# Shows: power.W, GFX.%, VRAM used, temperatures

# NVIDIA/AMD/Intel monitor:
nvtop
```

### GPU power via perf

```bash
# AMD: perf event for power:
perf stat -e power/energy-gpu/ sleep 10
# Shows total GPU energy (Joules) over 10s

# Intel GPU energy:
perf stat -e "intel_gpu/energy/" sleep 10
```

---

## Integrations

- **Ch01 (DRM Architecture)** — runtime PM hooks (`runtime_suspend`/`runtime_resume`) are called from the DRM core when the GPU becomes idle; power management is part of the DRM driver lifecycle
- **Ch22 (RADV)** — RADV submits workloads that trigger AMD DPM to scale up GPU frequency; idle detection causes DPM to scale down
- **Ch23 (ANV)** — Intel's GuC power management is initialised as part of the i915/xe driver setup; RC6 residency affects GuC submission latency
- **Ch49 (Multi-GPU PRIME)** — Discrete GPUs on laptops use runtime PM to sleep when not needed; PRIME render offload wakes the dGPU on demand, adding ~10–30ms latency
- **Ch159 (Panfrost)** — Panfrost's devfreq integration uses simple_ondemand governor; thermal management is via the ARM thermal zone driver on SoCs
- **Ch160 (Turnip/Adreno)** — Qualcomm Adreno uses the GMU for autonomous power management; the kernel driver communicates with GMU via the HFI (Host Firmware Interface) protocol
