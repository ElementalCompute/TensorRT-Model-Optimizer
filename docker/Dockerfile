FROM nvcr.io/nvidia/pytorch:25.06-py3

ARG PIP_EXTRA_INDEX_URL="https://pypi.nvidia.com"
ARG TRT_LLM_COMMIT=v1.0.0rc0
ARG REMOVE_TRT_LLM_SRC=0
ARG CUDA_ARCH="89-real;100-real;120-real"

ENV PIP_EXTRA_INDEX_URL=$PIP_EXTRA_INDEX_URL \
    PIP_NO_CACHE_DIR=off \
    PIP_CONSTRAINT= \
    TORCH_CUDA_ARCH_LIST="8.9;10.0;12.0"

WORKDIR /workspace

# Install TensorRT-LLM from source
RUN --mount=type=ssh,id=nvidia git clone https://github.com/NVIDIA/TensorRT-LLM.git tensorrt-llm \
    && cd tensorrt-llm \
    && git checkout ${TRT_LLM_COMMIT} \
    && git submodule update --init --recursive \
    # Patch SM120 fuse_fp4 guard and moe gemm
    && sed -ri \
       -e 's/!mFuseFp4Quant[[:space:]]*\|\|[[:space:]]*mSM[[:space:]]*==[[:space:]]*100/!mFuseFp4Quant || mSM >= 100/' \
       -e 's/fuse_fp4_quant only supports SM100 devices\./fuse_fp4_quant only supports SM100+ devices./' \
       cpp/tensorrt_llm/common/attentionOp.cpp \
    && sed -ri \
       -e 's/sm_ < 120/sm_ <= 120/' \
       cpp/tensorrt_llm/kernels/cutlass_kernels/moe_gemm/moe_gemm_template_dispatch.h

# Install required dependencies
RUN bash tensorrt-llm/docker/common/install_base.sh $(python --version 2>&1 | awk '{print $2}')
RUN bash tensorrt-llm/docker/common/install_cmake.sh
RUN bash tensorrt-llm/docker/common/install_mpi4py.sh
RUN bash tensorrt-llm/docker/common/install_tensorrt.sh
RUN bash tensorrt-llm/docker/common/install_cuda_toolkit.sh

RUN cd tensorrt-llm && git lfs install && git lfs pull

RUN cd tensorrt-llm \
    && ./scripts/build_wheel.py --job_count $(nproc) --clean --python_bindings --benchmarks --install --cuda_architecture=${CUDA_ARCH} \
    && git rev-parse --short HEAD > /workspace/tensorrt-llm.commit \
    && chmod -R 777 .
RUN pip install tensorrt-llm/build/tensorrt_llm*.whl

# Remove TensorRT-LLM source code to reduce image size except for benchmarks and examples folders
RUN if [ "$REMOVE_TRT_LLM_SRC" = "1" ]; then \
    mkdir -p tensorrt-llm_keep; \
    mv tensorrt-llm/benchmarks tensorrt-llm_keep/benchmarks; \
    mv tensorrt-llm/examples tensorrt-llm_keep/examples; \
    rm -rf tensorrt-llm; \
    mv tensorrt-llm_keep tensorrt-llm; \
    fi

# Update PATH and LD_LIBRARY_PATH variables for the TensorRT binaries
ENV LD_LIBRARY_PATH="/usr/local/tensorrt/targets/x86_64-linux-gnu/lib:${LD_LIBRARY_PATH}" \
    PATH="/usr/local/tensorrt/targets/x86_64-linux-gnu/bin:${PATH}"

# Export the path to 'libcudnn.so.X' needed by 'libonnxruntime_providers_tensorrt.so'
ENV LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH

# Install modelopt with all optional dependencies and pre-compile CUDA extensions otherwise they take several minutes on every docker run
RUN pip install -U "nvidia-modelopt[all,dev-test]"
RUN python -c "import modelopt.torch.quantization.extensions as ext; ext.precompile()"

# Find and install requirements.txt files for all examples excluding windows
COPY . TensorRT-Model-Optimizer
RUN rm -rf TensorRT-Model-Optimizer/.git
RUN find TensorRT-Model-Optimizer/examples -name "requirements.txt" | grep -v "windows" | while read req_file; do \
        echo "Installing from $req_file"; \
        pip install -r "$req_file" || exit 1; \
    done

# BNB CUDA 12.9
RUN pip uninstall -y bitsandbytes && \
    git clone https://github.com/bitsandbytes-foundation/bitsandbytes.git && \
    cd bitsandbytes && \
    cmake -S . -DCOMPUTE_BACKEND=cuda -DCOMPUTE_CAPABILITY="89;90;100;120" && \
    make -j$(nproc) && \
    pip install .  --no-deps

# Allow users to run without root
RUN chmod -R 777 /workspace
