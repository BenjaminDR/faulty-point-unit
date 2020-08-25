# Faulty Point Unit: ABI Poisoning Attacks on Intel SGX

This repository collects source code and data to reproduce the research published in our paper "Faulty Point Unit: ABI Poisoning Attacks on Intel SGX" to appear at [ACSAC'20](https://www.acsac.org/).

## Abstract

This paper analyzes a previously overlooked attack surface that allows unprivileged adversaries to impact supposedly secure floating-point computations in Intel SGX enclaves through the Application Binary Interface (ABI). In a comprehensive study across 7 widely used industry-standard and research enclave shielding runtimes, we show that control and state registers of the x87 Floating-Point Unit(FPU) and Intel Streaming SIMD Extensions (SSE) are not always properly sanitized on enclave entry. First, we abuse the adversary's control over precision and rounding modes as a novel "ABI-level fault injection" primitive to silently corrupt enclaved floating-point operations, enabling a new class of stealthy, integrity-only attacks that disturb the result of SGX enclave computations. Our analysis reveals that this threat is especially relevant for applications that use the older x87 FPU, which is still being used under certain conditions for high-precision operations by modern compilers like `gcc`. We exemplify the potential impact of ABI-level quality-degradation attacks in a case study of an enclaved machine learning service and in a larger analysis on the SPEC benchmark programs. Second,we explore the impact on enclave confidentiality by showing that the adversary's control over floating-point exception masks can be abused as an innovative controlled channel to detect FPU usage and to recover enclaved multiplication operands in certain scenarios. Our findings, affecting 5 out of the 7 studied runtimes, demonstrate the fallacy and challenges of implementing high-assurance trusted execution environments on contemporary x86 hardware. We responsibly disclosed our findings to the vendors and were assigned two CVEs, leading to patches in the Intel SGX-SDK, Microsoft OpenEnclave, and the Rust compiler's SGX target.

```
@inproceedings{alder2020faulty,
    title     = {Faulty Point Unit: {ABI} Poisoning Attacks on {Intel SGX}},
    author    = {Alder, Fritz and Van Bulck, Jo and Oswald, David and Piessens, Frank},
    booktitle = {36th Annual Computer Security Applications Conference {(ACSAC)}},
    month     = Dec,
    year      = 2020,
}
```

## Artifact description

In this artifact, we provide the means to reproduce all of our experimental results. More specifically, the repository is organized following the tables and figures in the paper:

| Paper reference | Directory | Description |
| :---- | :-------- | :---------- |
| Table 1 | `01_table1_basic-poc` | Proof-of-concept poisoning attack executed against an Intel SGX-SDK enclave. We provide the documented source code, build instructions, and the raw paper data.|
| Table 2 | `02_table2_affected_runtimes` | List of affected enclave shielding runtimes. We provide proof-of-concept poisoning attacks against all runtimes marked with a star. Namely, OpenEnclave, SGX-LKL, Rust-EDP, and Go-TEE (for Intel SGX-SDK, see the artifact for Table 1).|
| Figure 4 | `03_section-4_controlled-channel` | Controlled channel attack. We provide the documented proof-of-concept attacker and victim source code, build instructions, and the raw paper data.|
| Table 3 | `04_section-5_mnist-javascript` | Machine learning with Javascript inside an enclave. We provide the documented source code, build instructions, and the raw paper data. This example was built on the open-source code of the S-FaaS project.|
| Table 4 | `05_section-6_spec-cpu-2017` | Benchmarks with SPEC CPU 2017. Due to licensing and copyright restrictions with the proprietary SPEC 2017 suite, we can only provide the used configuration file and detailed instructions how to reproduce our results for people who already bought the SPEC CPU 2017 suite. Please note however, that obtaining the raw data for the SPEC benchmarks summarized in Table 4 takes several CPU weeks. For the artifact evaluation, we therefore do *not* expect artifact reviewers to generate the raw data themselves. We instead provide the log outputs of the full SPEC runs with detailed instructions of how to reproduce the summary in Table 4.|
| Figure 6 | `06_figure-6_blender-outputs` | Screenshot of the Blender benchmark. We provide the full reference output of Blender in normal conditions and under the attack.|

## Preparation and setup

Detailed instructions to reproduce the above artifacts are provided in the READMEs of the respective subdirectories. In this section, we first overview the general setup common to all the experiments.

### Building the Intel SGX simulator

All of the attacks described in our paper were performed on *real SGX hardware*. For the convenience of the artifact evaluation however, we ensured that all of our main results can also be ran in the official Intel SGX simulator, i.e., on normal processors without Intel SGX support. To easily run the simulator, we further provide an installation script and a Docker container with all the dependencies preinstalled.

**Note (simulation validity).** Simulation mode does assuredly _not_ affect the validity of our attacks, as we exploit software sanitization oversights that occur in code that is agnostic to whether the enclave executes on real hardware or in a simulator.

**Note (hardware access).** Unfortunately not all of the enclave shielding runtimes that we studied (e.g., Rust-EDP and Go-TEE) offer a simulation mode. We are, hence, not able to offer a simulation mode for attacks on runtimes other than the Intel SGX-SDK. This is only a concern for Table 2 (`02_table2_affected_runtimes`). To test the artifacts in a real hardware environment, we will provide ACSAC artifact reviewers with SSH access to a preinstalled SGX-enabled lab machine.

The following is a quick Docker installation guide (Debian GNU/Linux and Ubuntu), loosely based on
https://docs.docker.com/get-started/

#### 1. Install docker

```bash
$ sudo apt-get install docker.io
```

#### 2. Configure user access

```bash
$ sudo usermod -aG docker $(whoami) # add users to docker group, then re-login.
```

#### 3. Check your docker installation

```bash
$ docker run hello-world
[...]
This message shows that your installation appears to be working correctly.
[...]
```

#### 4. Build and run the 'sgx-fpu' Docker image

```bash
$ make build
$ make run 
docker run -i -h "badf1oa7" -t sgx-fpu 
========================================================================
= Faulty Point Unit SGX simulator Docker container                     =
========================================================================
Description:	Ubuntu 18.04.4 LTS

To get started, see README.md in the current directory

root@badf1oa7:/faulty-point-unit#
```

### Switching Intel SGX-SDK versions

This artifact demonstrates an attack that existed in the Intel SGX-SDK in version 2.7.1 and was patched following with version 2.8.0. As such, we provide an easy to use install script that allows to reproduce the attack in both versions and be able to verify its mitigation in the patched SDK version.

We provide the script `sdk_helper.sh` to install and switch between the two relevant Intel SGX-SDK versions. Simply run `./sdk_helper.sh install` to install the relevant dependencies in hardware mode or `./sdk_helper install_sim` to install the dependencies in simulation mode if you have no intention of running the attack on real hardware. As described earlier, since this attack exploits software sanitization oversights, the simulation yields the same results when reproducing the data as real hardware will.

The `sdk_helper.sh` script has two options to switch between Intel SGX-SDK versions: `sdk_helper.sh vulnerable` and `sdk_helper.sh vulnerable` to switch to the Intel SGX-SDK 2.7.1 and 2.8 respectively. **Note** that in order for the script to perform this switch reliably, this script needs to be called with `source` so that the environment variables that are set are inherited by your shell. As such, always call the script as `source sdk_helper.sh vulnerable` and `source sdk_helper.sh vulnerable`!

A quick start example of commands to start with this artifact is as follows:
```bash
# Install dependencies
./sdk_helper.sh install_sim
# Switch to 2.7.1
source ./sdk_helper.sh vulnerable
# Reproduce basic proof of concept attack
cd 01_table1_basic-poc
./eval_simulator.sh
# Now reproduce same attack with patched SDK
source ../sdk_helper.sh patched
./eval_simulator.sh
```

Use `./sdk_helper.sh help` to print an overview of commands and tasks the script performs.

## Intel SGX-SDK bug in Ubuntu 20.04LTS
As of release of this artifact, there is a known issue with the Intel SGX-SDK and Ubuntu LTS 20.04 as reported [here](https://lore.kernel.org/linux-sgx/4db41057-910c-b686-0428-474debe382c1@fortanix.com/), [here](https://github.com/intel/linux-sgx/issues/569), and [here](https://github.com/intel/linux-sgx/pull/515). Since we are deliberately installing an obsolete Intel SGX-SDK to make it vulnerable to a known attack, we can not rely on future releases to patch this issue. A quick workaround is to set `mount -o remount,exec /dev` to circumvent the new behavior of Ubuntu 20.04. The `sdk_helper` sets this before installing if it detects the mentioned Ubuntu version. We **explicitly** warn of possible side-effects. Additionally, since this setting resets on reboot, a user might have to simply rerun the installation with the `sdk_helper` after a reboot.