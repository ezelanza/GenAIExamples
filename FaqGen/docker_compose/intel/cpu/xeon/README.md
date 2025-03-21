# Build Mega Service of FAQ Generation on Intel Xeon Processor

This document outlines the deployment process for a FAQ Generation application utilizing the [GenAIComps](https://github.com/opea-project/GenAIComps.git) microservice pipeline on an Intel Xeon server. The steps include Docker image creation, container deployment via Docker Compose, and service execution to integrate microservices such as `llm`. We will publish the Docker images to Docker Hub soon, which will simplify the deployment process for this service.

## 🚀 Apply Intel Xeon Server on AWS

To apply a Intel Xeon server on AWS, start by creating an AWS account if you don't have one already. Then, head to the [EC2 Console](https://console.aws.amazon.com/ec2/v2/home) to begin the process. Within the EC2 service, select the Amazon EC2 M7i or M7i-flex instance type to leverage 4th Generation Intel Xeon Scalable processors. These instances are optimized for high-performance computing and demanding workloads.

For detailed information about these instance types, you can refer to this [link](https://aws.amazon.com/ec2/instance-types/m7i/). Once you've chosen the appropriate instance type, proceed with configuring your instance settings, including network configurations, security groups, and storage options.

After launching your instance, you can connect to it using SSH (for Linux instances) or Remote Desktop Protocol (RDP) (for Windows instances). From there, you'll have full access to your Xeon server, allowing you to install, configure, and manage your applications as needed.

## 🚀 Build Docker Images

First of all, you need to build Docker Images locally. This step can be ignored once the Docker images are published to Docker hub.

### 1. Build vLLM Image

```bash
git clone https://github.com/vllm-project/vllm.git
cd ./vllm/
VLLM_VER="$(git describe --tags "$(git rev-list --tags --max-count=1)" )"
git checkout ${VLLM_VER}
docker build --no-cache --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f Dockerfile.cpu -t opea/vllm:latest --shm-size=128g .
```

### 2. Build LLM Image

```bash
git clone https://github.com/opea-project/GenAIComps.git
cd GenAIComps
docker build -t opea/llm-faqgen:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/llms/src/faq-generation/Dockerfile .
```

### 3. Build MegaService Docker Image

To construct the Mega Service, we utilize the [GenAIComps](https://github.com/opea-project/GenAIComps.git) microservice pipeline within the `faqgen.py` Python script. Build the MegaService Docker image via below command:

```bash
git clone https://github.com/opea-project/GenAIExamples
cd GenAIExamples/FaqGen/
docker build --no-cache -t opea/faqgen:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f GenAIExamples/FaqGen/Dockerfile .
```

### 4. Build UI Docker Image

Build the frontend Docker image via below command:

```bash
cd GenAIExamples/FaqGen/ui
docker build -t opea/faqgen-ui:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f docker/Dockerfile .
```

### 5. Build react UI Docker Image (Optional)

Build the frontend Docker image based on react framework via below command:

```bash
cd GenAIExamples/FaqGen/ui
export BACKEND_SERVICE_ENDPOINT="http://${host_ip}:8888/v1/faqgen"
docker build -t opea/faqgen-react-ui:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy --build-arg BACKEND_SERVICE_ENDPOINT=$BACKEND_SERVICE_ENDPOINT -f docker/Dockerfile.react .
```

Then run the command `docker images`, you will have the following Docker Images:

1. `opea/vllm:latest`
2. `opea/llm-faqgen:latest`
3. `opea/faqgen:latest`
4. `opea/faqgen-ui:latest`
5. `opea/faqgen-react-ui:latest`

## 🚀 Start Microservices and MegaService

### Required Models

We set default model as "meta-llama/Meta-Llama-3-8B-Instruct", change "LLM_MODEL_ID" in following Environment Variables setting if you want to use other models.

If use gated models, you also need to provide [huggingface token](https://huggingface.co/docs/hub/security-tokens) to "HUGGINGFACEHUB_API_TOKEN" environment variable.

### Setup Environment Variables

Since the `compose.yaml` will consume some environment variables, you need to setup them in advance as below.

```bash
export no_proxy=${your_no_proxy}
export http_proxy=${your_http_proxy}
export https_proxy=${your_http_proxy}
export host_ip=${your_host_ip}
export LLM_ENDPOINT_PORT=8008
export LLM_SERVICE_PORT=9000
export FAQGEN_BACKEND_PORT=8888
export FAQGen_COMPONENT_NAME="OpeaFaqGenvLLM"
export LLM_MODEL_ID="meta-llama/Meta-Llama-3-8B-Instruct"
export HUGGINGFACEHUB_API_TOKEN=${your_hf_api_token}
export MEGA_SERVICE_HOST_IP=${host_ip}
export LLM_SERVICE_HOST_IP=${host_ip}
export LLM_ENDPOINT="http://${host_ip}:${LLM_ENDPOINT_PORT}"
export BACKEND_SERVICE_ENDPOINT="http://${host_ip}:8888/v1/faqgen"
```

Note: Please replace with `your_host_ip` with your external IP address, do not use localhost.

### Start Microservice Docker Containers

```bash
cd GenAIExamples/FaqGen/docker_compose/intel/cpu/xeon
docker compose up -d
```

### Validate Microservices

1. vLLM Service

```bash
curl http://${host_ip}:${LLM_ENDPOINT_PORT}/v1/chat/completions \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"model": "meta-llama/Meta-Llama-3-8B-Instruct", "messages": [{"role": "user", "content": "What is Deep Learning?"}]}'
```

2. LLM Microservice

```bash
curl http://${host_ip}:${LLM_SERVICE_PORT}/v1/faqgen \
  -X POST \
  -d '{"query":"Text Embeddings Inference (TEI) is a toolkit for deploying and serving open source text embeddings and sequence classification models. TEI enables high-performance extraction for the most popular models, including FlagEmbedding, Ember, GTE and E5."}' \
  -H 'Content-Type: application/json'
```

3. MegaService

```bash
curl http://${host_ip}:${FAQGEN_BACKEND_PORT}/v1/faqgen \
  -H "Content-Type: multipart/form-data" \
  -F "messages=Text Embeddings Inference (TEI) is a toolkit for deploying and serving open source text embeddings and sequence classification models. TEI enables high-performance extraction for the most popular models, including FlagEmbedding, Ember, GTE and E5." \
  -F "max_tokens=32" \
  -F "stream=False"
```

```bash
## enable stream
curl http://${host_ip}:${FAQGEN_BACKEND_PORT}/v1/faqgen \
  -H "Content-Type: multipart/form-data" \
  -F "messages=Text Embeddings Inference (TEI) is a toolkit for deploying and serving open source text embeddings and sequence classification models. TEI enables high-performance extraction for the most popular models, including FlagEmbedding, Ember, GTE and E5." \
  -F "max_tokens=32" \
  -F "stream=True"
```

Following the validation of all aforementioned microservices, we are now prepared to construct a mega-service.

## 🚀 Launch the UI

Open this URL `http://{host_ip}:5173` in your browser to access the frontend.

![project-screenshot](../../../../assets/img/faqgen_ui_text.png)

## 🚀 Launch the React UI (Optional)

To access the FAQGen (react based) frontend, modify the UI service in the `compose.yaml` file. Replace `faqgen-xeon-ui-server` service with the `faqgen-xeon-react-ui-server` service as per the config below:

```bash
  faqgen-xeon-react-ui-server:
    image: opea/faqgen-react-ui:latest
    container_name: faqgen-xeon-react-ui-server
    environment:
      - no_proxy=${no_proxy}
      - https_proxy=${https_proxy}
      - http_proxy=${http_proxy}
    ports:
      - 5174:80
    depends_on:
      - faqgen-xeon-backend-server
    ipc: host
    restart: always
```

Open this URL `http://{host_ip}:5174` in your browser to access the react based frontend.

- Create FAQs from Text input
  ![project-screenshot](../../../../assets/img/faqgen_react_ui_text.png)

- Create FAQs from Text Files
  ![project-screenshot](../../../../assets/img/faqgen_react_ui_text_file.png)
