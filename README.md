# Edge AI における CI/CD を GitHub で構成するデモ

Configure with Azure IoT Edge, Custom Vision in Azure Cognitive Services, and GitHub Actions


## 資料


### Azure Custom Vision によるイテレーション（モデル）のエクスポート

- [Custom Vision とは - Azure Cognitive Services | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/cognitive-services/custom-vision-service/overview)
- [モデルをモバイルにエクスポートする - Custom Vision Service - Azure Cognitive Services | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/cognitive-services/custom-vision-service/export-your-model)
- [プログラムを使用してモデルをエクスポートする - Azure Cognitive Services | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/cognitive-services/custom-vision-service/export-programmatically)
- [Custom Vision REST API reference - Azure Cognitive Services | Microsoft Learn](https://learn.microsoft.com/en-us/rest/api/custom-vision/)
  - [Export Iteration - Export Iteration](https://learn.microsoft.com/en-us/rest/api/customvision/training3.3/export-iteration/export-iteration?tabs=HTTP)


### Cognitive Services における CI/CD

- [Azure Cognitive Services の開発オプション - Azure Cognitive Services | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/cognitive-services/cognitive-services-development-options)


### Azure IoT Edge

- [Deploy modules from the Azure CLI command line - Azure IoT Edge | Microsoft Learn](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-deploy-modules-cli?view=iotedge-1.4)
- [Deploy modules at scale using Azure CLI - Azure IoT Edge | Microsoft Learn](https://learn.microsoft.com/en-us/azure/iot-edge/how-to-deploy-cli-at-scale?view=iotedge-1.4)
- [Deploy module & routes with deployment manifests - Azure IoT Edge | Microsoft Learn](https://learn.microsoft.com/en-us/azure/iot-edge/module-composition?view=iotedge-1.4)


### 分類用の画像データセット

- [Food-101 -- Mining Discriminative Components with Random Forests](https://data.vision.ee.ethz.ch/cvl/datasets_extra/food-101/)
- [Open Images V7](https://storage.googleapis.com/openimages/web/index.html)


## 検証メモ

### Custom Vision のエクスポート

#### 手組での作業ログ

```bash
CUSTOM_VISION_ENDPOINT=
CUSTOM_VISION_PROJECT_ID=
CUSTOM_VISION_ITERATION_ID=
CUSTOM_VISION_EXPORT_PLATFORM="Dockerfile"
CUSTOM_VISION_EXPORT_FLAVOR="Linux"

# Export an iteration
curl -X POST -i -H "training-key: ${CUSTOM_VISION_TRAINING_KEY}" -H "content-length: 0" "https://${CUSTOM_VISION_ENDPOINT}/customvision/v3.3/training/projects/${CUSTOM_VISION_PROJECT_ID}/iterations/${CUSTOM_VISION_ITERATION_ID}/export?platform=${CUSTOM_VISION_EXPORT_PLATFORM}&flavor=${CUSTOM_VISION_EXPORT_FLAVOR}"

# Get an export
EXPORTS=$(curl -X GET -H "training-key: ${CUSTOM_VISION_TRAINING_KEY}" "https://${CUSTOM_VISION_ENDPOINT}/customvision/v3.3/training/projects/${CUSTOM_VISION_PROJECT_ID}/iterations/${CUSTOM_VISION_ITERATION_ID}/export")

# Download the export
DOWNLOAD_URI=$(echo ${EXPORTS} | jq -r ".[] | select(.platform == \"DockerFile\") | .downloadUri")
FILE_NAME=$(echo ${DOWNLOAD_URI) | sed -r "s/.*\/([^\/]*.zip)\?.*/\\1/g")
curl -o ${FILE_NAME} ${DOWNLOAD_URI}
unzip ${FILE_NAME} -d exported
```


### IoT Edge によるエッジへのアプリケーション配布


#### 手組みでの作業ログ

```bash
PAT_FOR_CONTAINER_REGISTRY=
CUSTOM_VISION_MODEL_VERSION=
IOT_HUB_NAME=
IOT_HUB_DEVICE_ID=
DEPLOYMENT_ID=

mkdir deployment
pushd deployment

DEPLOYMENT_JSON=$(cat ../iot-edge/deployment.json)
DEPLOYMENT_JSON=$(echo $DEPLOYMENT_JSON | jq ".content.modulesContent.\"\$edgeAgent\".\"properties.desired\".runtime.settings.registryCredentials.ghcr.password=\"${PAT_FOR_CONTAINER_REGISTRY}\"")
DEPLOYMENT_JSON=$(echo $DEPLOYMENT_JSON | jq ".content.modulesContent.\"\$edgeAgent\".\"properties.desired\".modules.\"custom-vision\".settings.image=\"ghcr.io/dzeyelid/custom-vision:${CUSTOM_VISION_MODEL_VERSION}\"")
echo $DEPLOYMENT_JSON | jq > iot-edge_deployment.json

az login

# 初回
az iot edge deployment create --deployment-id ${DEPLOYMENT_ID} --hub-name ${IOT_HUB_NAME} --content ./iot-edge_deployment.json

# 更新
az iot edge deployment create --deployment-id ${DEPLOYMENT_ID} --hub-name ${IOT_HUB_NAME} --content ./iot-edge_deployment.json

popd
```

