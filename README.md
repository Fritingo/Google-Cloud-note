# Google-Cloud-note
learn google cloud 

*Cloud Shell*

#### set project
`gcloud config set project <PROJECT ID>`

#### create cloud storage bucket
`gsutil mb -c regional -l us-central1  gs://<bucket name>`
region:us-central1-a

#### create BigQuery dataset
`bq mk <dataset name>`

#### connect instance SSH
`gcloud compute ssh <instance> --zone us-central1-a`