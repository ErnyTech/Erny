---
title: "iOS IPA Signing Service"
---

This is a private IPA signing service, it is not under credential protection as UDIDs are unique.

<form id="signForm">
  <label for="ipaFile">Upload IPA File:</label>
  <input type="file" id="ipaFile" name="file" accept=".ipa" style="display: none;">
  <label for="ipaFile" id="ipaFileLabel" class="custom-file-label">Choose file</label>
  <br><br>
  <label for="udid">Device UDID:</label>
  <input type="password" id="udid" name="udid" placeholder="Enter your device UDID" required autocomplete="current-password" class="form-input">
  <br><br>
  <button type="submit" class="text-button">Sign IPA</button>
</form>

<br>
<div class="loader" id="loader"></div>
    
<div class="progress-container" id="progressContainer">
  <label for="progressFill">Uploading:</label>
  <div class="custom-progress-bar">
    <div class="custom-progress-fill" id="progressFill"></div>
  </div>
  <span id="progressPercent">0%</span>
</div>
  
<div id="message" class="message"></div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
<script>
  const form = document.getElementById('signForm');
  const messageDiv = document.getElementById('message');
  const loader = document.getElementById('loader');
  const progressContainer = document.getElementById('progressContainer');
  const progressFill = document.getElementById('progressFill');
  const progressPercent = document.getElementById('progressPercent');
        
  let currentProgress = 0;
  let targetProgress = 0;
        
  const isAppleDevice = () => {
    return /iPhone|iPad|iPod|AppleWatch|Vision/i.test(navigator.userAgent);
  };
        
  document.getElementById("ipaFile").addEventListener("change", function(event) {
    if (event.target.files.length > 0) {
      document.getElementById("ipaFileLabel").textContent = event.target.files[0].name;
    }
  });
        
  form.addEventListener('submit', function (e) {
    e.preventDefault();
    messageDiv.style.display = 'none';
    loader.style.display = 'block';
    progressContainer.style.display = 'block';
    progressFill.style.width = '0%';
    progressPercent.textContent = '0%';
    messageDiv.innerHTML = '';
    currentProgress = 0;
    targetProgress = 0;
            
    const formData = new FormData(form);
    const xhr = new XMLHttpRequest();
            
    xhr.open('POST', 'https://iosign.erny.dev/sign', true);
            
    xhr.upload.addEventListener('progress', (event) => {
      if (event.lengthComputable) {
        if (event.loaded === event.total) {
          progressContainer.style.display = 'none';
          messageDiv.classList.remove('error', 'success');
          messageDiv.classList.add('success');
          messageDiv.innerHTML = `
            <strong>IPA signing is in progress. Please wait...</strong>
          `;
          messageDiv.style.display = 'block';
        } else {
          targetProgress = Math.round((event.loaded / event.total) * 100);
          animateProgress();
        }
      } else {
        console.log('Progress not computable');
      }
    });
            
    xhr.onreadystatechange = function () {
      if (xhr.readyState === XMLHttpRequest.DONE) {
        console.log('Upload completed');
        loader.style.display = 'none';
        progressContainer.style.display = 'none';
        
        if (xhr.status === 200) {
          const result = JSON.parse(xhr.responseText);
          messageDiv.classList.add('success');

          messageDiv.innerHTML = `
            <strong>${result.message}</strong>
            <div class="links">
              <a href="${result.ipa_url}" target="_blank">Download Signed IPA</a>
              <br>
              <a href="${result.ota_url}" target="_blank">Download OTA Plist</a>
            </div>
            <br>
          `;
          
          if (isAppleDevice()) {
            messageDiv.innerHTML += `
              <a href="${result.install_url}" class="install-button">Install on Device</a>
            `;
          } else {
            var qrSize = Math.floor(window.innerWidth * 0.15);
            messageDiv.innerHTML += `
              <strong>Scan this QR code with the iOS Camera app to install the IPA</strong>
              <br><br>
              <div id="qrCode" style="width: ${qrSize}px; height: ${qrSize}px;"></div>
            `;
            
            new QRCode(document.getElementById("qrCode"), {
              text: result.install_url,
              width: qrSize,
              height: qrSize,
              colorDark: "#000000",
              colorLight: "#ffffff",
              correctLevel: QRCode.CorrectLevel.M
            });
          }
          
          messageDiv.style.display = 'block';
        } else {
          const result = JSON.parse(xhr.responseText);
          messageDiv.classList.add('error');
          messageDiv.textContent = result.message || 'An error occurred while processing your request.';
          messageDiv.style.display = 'block';
        }
      }
    };
            
    xhr.onerror = function () {
      console.log('Network error occurred');
      loader.style.display = 'none';
      progressContainer.style.display = 'none';
      messageDiv.classList.add('error');
      messageDiv.textContent = 'A network error occurred. Please try again later.';
      messageDiv.style.display = 'block';
    };
            
    xhr.send(formData);
  });
        
  function animateProgress() {
    if (currentProgress < targetProgress) {
      currentProgress++;
      progressFill.style.width = `${currentProgress}%`;
      progressPercent.textContent = `${currentProgress}%`;
      requestAnimationFrame(animateProgress);
    }
  }
</script>
