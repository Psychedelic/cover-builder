![](https://docs.covercode.ooo/overview/imgs/mainn.png)

# Cover builder

Cover builder is a key component of [Cover](https://covercode.ooo/), the open internet service that helps build and
verify the code of canisters on the Internet Computer.

- [Main Repo](https://github.com/Psychedelic/cover/)
- [Cover Validator](https://github.com/Psychedelic/cover-validator/)

## How it works

- You will build your wasm and hash it with the Dockerfile we provided to make sure we have the same environment and the
  hash will come out deterministic.
- Then you will provide us with the config you used to build your wasm, these inputs will be validated by Cover-validate
  before Cover proceeds to build or save your config.
- You can log in with Plug and build your saved configs.
- Cover will create a verification for your build result. The verification will contain information about your build
  like build status and build URL for you to see the build process.
- The hash Cover built will be compared with the one on network IC

## Requirement

- **_Cover only support DFX 0.8.4 or above_**
- **NOTE**: If you're using _DFX 0.9.3_ and above, when you build your wasm, DFX might automatically optimize the wasm for you when `ic-cdk-optimizer` is installed.
- Same environment with our `Builder` ([dockerfile](https://github.com/Psychedelic/cover-builder/blob/main/dockerfile)).
- Note that `dfx version` or `rust version` can affect the build result.
- Must specify `type` field in the `dfx.json` file, we only support `rust`, `motoko` and `custom`.
- **NOTE**: _DFX 0.11.0_ and above will run optimization automatically for `rust` ([reference](https://internetcomputer.org/docs/current/developer-docs/updates/release-notes/#refactor-optimize-from-ic-wasm)).
- Cover validator and builder will update the build status for you to follow. You can only re-build your canister when
  the Cover builder finishes its job and updates the status to either Error or Success. If the Cover builder failed to
  update build status, you’ll have to wait **5 minutes** before rebuilding your canister. So make sure to fill your
  inputs correctly.

  Example:

```json
{
  "dfx": "0.8.4",
  "canisters": {
    "cover": {
      "candid": "cover.did",
      "type": "rust",
      "package": "cover"
    }
  }
}
```

- Must have `canister_ids.json` at root directory

Example:

```json
{
  "cover": {
    "ic": "iftvq-niaaa-aaaai-qasga-cai"
  }
}
```

## Usage

- Add this Dockerfile to your repo

```dockerfile
FROM --platform=linux/amd64 ubuntu:20.04
# Install a basic environment needed for our build tools
ARG DEBIAN_FRONTEND=noninteractive
RUN \
    apt -yq update && \
    apt -yqq install --no-install-recommends curl ca-certificates \
        build-essential pkg-config libssl-dev llvm-dev liblmdb-dev clang cmake git

# Replace your Rust version here
ARG rust_version=1.58.1
ENV RUSTUP_HOME=/opt/rustup \
    CARGO_HOME=/opt/cargo \
    PATH=/opt/cargo/bin:$PATH
RUN curl --fail https://sh.rustup.rs/ -sSf \
        | sh -s -- -y --default-toolchain ${rust_version}-x86_64-unknown-linux-gnu --no-modify-path && \
    rustup default ${rust_version}-x86_64-unknown-linux-gnu && \
    rustup target add wasm32-unknown-unknown
RUN cargo install ic-cdk-optimizer

# Install dfx; the version is picked up from the DFX_VERSION environment variable
# Replace your dfx version here
ENV DFX_VERSION=0.8.4
RUN sh -ci "$(curl -fsSL https://sdk.dfinity.org/install.sh)"

# COPY . /canister
# OR mount volumne

WORKDIR /canister

# Example: Build and Optimize
# RUN dfx build --network ic <your_canister_name>
# RUN ic-cdk-optimizer .dfx/ic/canisters/<your_canister_name>/<your_canister_name>.wasm \
#             -o .dfx/ic/canisters/<your_canister_name>/<your_canister_name>.wasm
# RUN openssl dgst -sha256 .dfx/ic/canisters/<your_canister_name>/<your_canister_name>.wasm | awk '/.+$/{print "0x"$2}' > wasm_hash
```

- Build docker image

```bash
$ docker build -t DOCKER_IMAGE_NAME -f DOCKERFILE_PATH .
```

- Run docker image and build your wasm

```bash
$ docker run -it DOCKER_IMAGE_NAME

# Bind mount example
$ docker run -it --mount type=bind,source="$(pwd)",target=/canister YOUR_CANISTER_NAME

# Build your canister
$ dfx build --network ic YOUR_CANISTER_NAME

# Run optimizer
$ ic-cdk-optimizer YOUR_IN_WASM.wasm -o YOUR_OUT_WASM.wasm

# Deploy
$ dfx canister --network ic install YOUR_CANISTER_NAME

# Check wasm hash on IC
$ dfx canister --network ic info YOUR_CANISTER_NAME

# To see your local wasm hash
$ openssl dgst -sha256 YOUR_WASM | awk '{print $2}'
```

- Check your Cover's verification
    - **Method 1**: Use Cover SDK
        - Document and example can be found [here](https://github.com/Psychedelic/cover-sdk)
    - **Method 2**: Go to [Cover site]().
        - To immediately build your config, choose Build then input the field correctly.
        - To build and save your config for later use, log in with Plug, choose Save build config, and inputs required
          fields. Save and choose a config to build
        - After cover builds your wasm, the hash and the status of the build process will be updated to your
          verification.
        - You can check out the Build URL field of the verification to see what went wrong with the build process if it
          failed.
    - **Method 3**: Use API.
        - You can use the API provided by [Cover-validator](https://github.com/Psychedelic/cover-validator) to validate
          and build your wasm
        - After that you can use this command to check your verification (it may take a while to build wasm and update
          your verification):

```bash
$  dfx canister --network ic call COVER_CANISTER_ID \
        getVerificationByCanisterId '(principal"YOUR_CANISTER_ID")'
```
