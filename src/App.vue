<template>
  <div>
    <input @change="upload" type="file" name="file" id="file" ref="file" />
    <button @click="abort">abort</button>
  </div>
</template>

<script setup>
import {
  S3Client,
  AbortMultipartUploadCommand,
  CreateMultipartUploadCommand,
  UploadPartCommand,
  CompleteMultipartUploadCommand,
} from "@aws-sdk/client-s3";

import { ref } from "vue";

const {
  VITE_REGION,
  VITE_ACCESS_KEY_ID,
  VITE_SECRET_ACCESS_KEY,
  VITE_BUCKET_NAME,
} = import.meta.env;

const fileName = ref("");
const uploadId = ref("");

const s3 = new S3Client({
  region: VITE_REGION,
  signatureVersion: "v3",
  credentials: {
    accessKeyId: VITE_ACCESS_KEY_ID,
    secretAccessKey: VITE_SECRET_ACCESS_KEY,
  },
});

const bucketName = VITE_BUCKET_NAME;
const expires = 20 * 60 * 1000;
const chunkSize = 10 * 1024 * 1024;

async function upload() {
  if (!event.target.files.length) {
    return;
  }
  console.log(event.target.files[0]);
  const file = event.target.files[0];

  fileName.value = file.name;

  const fileSize = file.size;

  const newDate = new Date().getTime();
  const datePlustwenty = new Date(newDate + expires);

  try {
    const createMultipartUpload = new CreateMultipartUploadCommand({
      Bucket: bucketName,
      Key: fileName.value,
      Expires: datePlustwenty,
    });

    const { UploadId } = await s3.send(createMultipartUpload);
    uploadId.value = UploadId;

    const chunkCount = Math.floor(fileSize / chunkSize) + 1;
    const uploadedChunks = [];

    for (let uploadCount = 1; uploadCount < chunkCount + 1; uploadCount++) {
      const start = (uploadCount - 1) * chunkSize;
      const end = uploadCount * chunkSize;
      const fileBlob =
        uploadCount < chunkCount ? file.slice(start, end) : file.slice(start);

      const uploadPart = new UploadPartCommand({
        Body: fileBlob,
        Bucket: bucketName,
        Key: fileName.value,
        PartNumber: uploadCount,
        UploadId,
      });

      const { ETag } = await s3.send(uploadPart);
      console.log("responseUploadPart", ETag);

      const uploadDetails = {
        ETag,
        PartNumber: uploadCount,
      };

      uploadedChunks.push(uploadDetails);
    }

    const completedMultipartUpload = new CompleteMultipartUploadCommand({
      Bucket: bucketName,
      Key: fileName.value,
      MultipartUpload: {
        Parts: uploadedChunks,
      },
      UploadId,
    });

    const { Location } = await s3.send(completedMultipartUpload);

    console.log("Location", Location);
  } catch (error) {
    console.log(error);
  }
}

async function abort() {
  const abortMultipartUploadCommand = new AbortMultipartUploadCommand({
    Bucket: bucketName,
    Key: fileName.value,
    UploadId: uploadId.value,
  });

  const data = await s3.send(abortMultipartUploadCommand);
  console.log(data);
  s3.destroy();
}
</script>
