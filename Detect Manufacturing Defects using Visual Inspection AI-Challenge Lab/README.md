# Detect Manufacturing Defects using Visual Inspection AI: Challenge Lab

lab: https://www.cloudskillsboost.google/focuses/34184?parent=catalog

### Task 1. Deploy the exported Cosmetic Inspection anomaly detection solution artifact

#### Navigation menu -> Compute Engine -> VM instances -> instance -> SSH

*SSH*

`
export DOCKER_TAG=gcr.io/ql-shared-resources-test/defect_solution@sha256:776fd8c65304ac017f5b9a986a1b8189695b7abbff6aa0e4ef693c46c7122f4c
`

`
export VISERVING_CPU_DOCKER_WITH_MODEL=${DOCKER_TAG}
export HTTP_PORT=8602
export LOCAL_METRIC_PORT=8603
`

`
docker pull ${VISERVING_CPU_DOCKER_WITH_MODEL}
`

`
docker run -v /secrets:/secrets --rm -d --name "product_inspection" \
--network="host" \
-p ${HTTP_PORT}:8602 \
-p ${LOCAL_METRIC_PORT}:8603 \
-t ${VISERVING_CPU_DOCKER_WITH_MODEL} \
--metric_project_id="${PROJECT_ID}" \
--service_account_credentials_json=/secrets/assembly-usage-reporter.json
`

`
docker container ls
`

### Task 2: Prepare resources to serve the exported assembly inspection solution artifact

*SSH*

`
gsutil cp gs://cloud-training/gsp895/prediction_script.py .
`

`
export PROJECT_ID=qwiklabs-gcp-01-f708fb518d2d
gsutil mb gs://${PROJECT_ID}
gsutil -m cp gs://cloud-training/gsp897/cosmetic-test-data/* gs://${PROJECT_ID}/cosmetic-test-data/
`

### Task 3: Identify a defective product image

*SSH*

`
gsutil cp gs://${PROJECT_ID}/cosmetic-test-data/IMG_07703.png .
`

`
python3 ./prediction_script.py --input_image_file=./IMG_07703.png  --port=8602 --output_result_file=defective_product.json
`

### Task 4: Identify a non-defective product

*SSH*

`
gsutil cp gs://${PROJECT_ID}/cosmetic-test-data/IMG_0769.png .
`

`
python3 ./prediction_script.py --input_image_file=./IMG_0769.png  --port=8602 --output_result_file=non_defective_product_result.json
`