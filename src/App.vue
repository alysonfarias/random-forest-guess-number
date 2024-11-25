<template>
  <div class="image-upload">
    <h2 class="title">Upload, Capture, or Select an Image</h2>

    <!-- File Upload Input -->
    <input type="file" accept="image/png, image/jpeg" @change="handleFileUpload" class="upload-input" />
    
    <!-- Capture from Camera -->
    <button @click="openCamera" class="camera-button">Take a Photo</button>
    <video v-if="cameraActive" ref="cameraStream" autoplay class="video-preview"></video>
    <button v-if="cameraActive" @click="capturePhoto" class="capture-button">Capture</button>

    <!-- Image Preview and Cropper -->
    <div v-if="imageUrl" class="crop-section">
      <h3 class="crop-title">Crop the Image</h3>
      <img ref="cropperImage" :src="imageUrl" alt="Crop Preview" class="crop-preview" />
      <button @click="cropImage" class="crop-button">Crop</button>
    </div>

    <!-- Cropped Image Preview -->
    <div v-if="croppedImageUrl" class="cropped-image-preview">
      <h3 class="cropped-title">Cropped Image Preview</h3>
      <img :src="croppedImageUrl" alt="Cropped Image" class="cropped-image" />
    </div>

    <!-- Submit Button -->
    <button v-if="croppedImage" @click="submitImage" class="submit-button">Submit</button>

    <!-- Display Prediction -->
    <div v-if="predicted_digit" class="prediction-section">
      <p class="prediction-text">Predicted Digit: {{ predicted_digit.predicted_digit }}</p>
      <img v-if="predicted_digit" :src="'data:image/png;base64,' + predicted_digit.processed_image" width="300px" class="processed-image" alt="Processed Image" />
    </div>
  </div>
</template>

<script>
import axios from "axios";
import Cropper from "cropperjs";
import "cropperjs/dist/cropper.css";

export default {
  data() {
    return {
      imageUrl: null,
      croppedImage: null,
      croppedImageUrl: null,
      cropper: null,
      predicted_digit: null,
      cameraActive: false,
    };
  },
  methods: {
    handleFileUpload(event) {
      const file = event.target.files[0];
      if (!file) return;
      if (!["image/png", "image/jpeg"].includes(file.type)) {
        alert("Only PNG and JPEG files are allowed.");
        return;
      }

      this.imageUrl = URL.createObjectURL(file);
      this.initializeCropper();
    },
    openCamera() {
      this.cameraActive = true;
      navigator.mediaDevices
        .getUserMedia({ video: true })
        .then((stream) => {
          const videoElement = this.$refs.cameraStream;
          videoElement.srcObject = stream;
        })
        .catch((err) => {
          console.error("Error accessing camera:", err);
          alert("Unable to access camera.");
        });
    },
    capturePhoto() {
      const videoElement = this.$refs.cameraStream;
      const canvas = document.createElement("canvas");
      canvas.width = videoElement.videoWidth;
      canvas.height = videoElement.videoHeight;
      const context = canvas.getContext("2d");
      context.drawImage(videoElement, 0, 0, canvas.width, canvas.height);
      
      // Stop the camera stream
      const stream = videoElement.srcObject;
      const tracks = stream.getTracks();
      tracks.forEach((track) => track.stop());
      
      this.cameraActive = false;

      // Set the captured image as the image source
      this.imageUrl = canvas.toDataURL("image/png");
      this.initializeCropper();
    },
    initializeCropper() {
      this.$nextTick(() => {
        const imageElement = this.$refs.cropperImage;
        if (this.cropper) {
          this.cropper.destroy();
        }
        this.cropper = new Cropper(imageElement, {
          aspectRatio: 1,
          viewMode: 1,
        });
      });
    },
    cropImage() {
      if (!this.cropper) return;

      // Get the cropped canvas while maintaining aspect ratio
      const canvas = this.cropper.getCroppedCanvas({
        aspectRatio: 1,  // Maintain aspect ratio
        width: 500,  // Resize to 500px wide
        height: 500  // Resize to 500px high
      });

      // Update the cropped image preview (Base64 URL for display)
      this.croppedImageUrl = canvas.toDataURL("image/png");

      // Convert the cropped canvas to Blob (for model prediction)
      canvas.toBlob((blob) => {
        this.croppedImage = blob; // Save the cropped image blob
      }, "image/png");
    },
    async submitImage() {
      if (!this.croppedImage) {
        alert("Please crop the image first.");
        return;
      }

      const formData = new FormData();
      formData.append("file", this.croppedImage, "cropped-image.png");

      try {
        const response = await axios.post(
          "https://number-guess-random-production.up.railway.app/predict/",
          formData,
          {
            headers: {
              "Content-Type": "multipart/form-data",
            },
          }
        );
        console.log("Response:", response.data);
        this.predicted_digit = response.data;
      } catch (error) {
        console.error("Error submitting image:", error);
        alert("Failed to submit image.");
      }
    },
  },
};
</script>

<style scoped>
.image-upload {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 30px;
  font-family: 'Arial', sans-serif;
  background: #f9f9f9;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
}

.title {
  font-size: 24px;
  color: #333;
  margin-bottom: 20px;
}

.upload-input {
  padding: 10px;
  margin-bottom: 20px;
  background-color: #ffffff;
  border: 1px solid #ddd;
  border-radius: 5px;
}

.camera-button,
.capture-button,
.crop-button,
.submit-button {
  padding: 10px 20px;
  margin: 10px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.camera-button:hover,
.capture-button:hover,
.crop-button:hover,
.submit-button:hover {
  background-color: #0056b3;
}

.video-preview {
  width: 100%;
  border-radius: 5px;
  margin-top: 20px;
}

.crop-section {
  margin-top: 20px;
  text-align: center;
}

.crop-preview {
  max-width: 100%;
  margin-top: 10px;
  border-radius: 10px;
}

.cropped-image-preview {
  margin-top: 20px;
  text-align: center;
}

.cropped-title {
  font-size: 20px;
  color: #333;
}

.cropped-image {
  max-width: 100%;
  margin-top: 10px;
  border-radius: 10px;
}

.prediction-section {
  margin-top: 20px;
  text-align: center;
}

.prediction-text {
  font-size: 18px;
  color: #333;
}

.processed-image {
  margin-top: 10px;
  max-width: 250px;
  border-radius: 10px;
}
</style>
