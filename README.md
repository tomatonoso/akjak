<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>秘密の写真</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background-color: #121212;
      color: #ffffff;
      padding-bottom: 30px;
    }
    header {
      text-align: center;
      padding: 16px;
      background-color: #1e1e1e;
      border-bottom: 1px solid #333;
    }
    h1 { font-size: 1.1rem; }
    main { padding: 15px; max-width: 500px; margin: 0 auto; }
    .controls { display: flex; gap: 10px; margin-bottom: 20px; }
    .btn {
      flex: 1;
      text-align: center;
      padding: 14px 8px;
      border-radius: 10px;
      font-weight: bold;
      font-size: 0.95rem;
      cursor: pointer;
      background-color: #0a84ff;
      color: white;
      display: block;
      user-select: none;
    }
    .video-btn { background-color: #ff453a; }
    .reset-btn {
      background-color: #3a3a3c;
      font-size: 0.8rem;
      padding: 8px;
      margin-top: 15px;
    }
    input[type="file"] { display: none; }
    .status-msg {
      text-align: center;
      font-size: 0.85rem;
      color: #8e8e93;
      margin-bottom: 15px;
    }
    .gallery {
      display: grid;
      grid-template-columns: repeat(2, 1fr);
      gap: 10px;
    }
    .media-card {
      position: relative;
      background-color: #1e1e1e;
      border-radius: 10px;
      overflow: hidden;
      aspect-ratio: 1 / 1;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .media-card img, .media-card video {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
    .delete-btn {
      position: absolute;
      top: 6px;
      right: 6px;
      background: rgba(0, 0, 0, 0.7);
      color: #ffffff;
      border: none;
      border-radius: 50%;
      width: 32px;
      height: 32px;
      font-size: 16px;
      line-height: 32px;
      text-align: center;
      cursor: pointer;
      z-index: 10;
    }
  </style>
</head>
<body>

  <header>
    <h1>秘密のアルバム</h1>
  </header>

  <main>
    <div class="controls">
      <label class="btn">
        📷 写真を撮影
        <input type="file" id="photoInput" accept="image/*" capture="environment">
      </label>
      <label class="btn video-btn">
        🎥 動画を撮影
        <input type="file" id="videoInput" accept="video/*" capture="environment">
      </label>
    </div>

    <div id="status" class="status-msg">起動中...</div>
    <div id="gallery" class="gallery"></div>

    <button class="btn reset-btn" onclick="resetDB()">⚠️ データベースを強制初期化</button>
  </main>

  <script>
    const statusEl = document.getElementById("status");
    const galleryEl = document.getElementById("gallery");
    const photoInput = document.getElementById("photoInput");
    const videoInput = document.getElementById("videoInput");

    let db = null;

    // バージョンを更新して自動移行
    const request = indexedDB.open("SecretMediaDB_v2", 1);

    request.onerror = (e) => {
      statusEl.textContent = "DBの起動に失敗しました（プライベートモード等の可能性があります）。";
    };

    request.onupgradeneeded = (e) => {
      db = e.target.result;
      if (!db.objectStoreNames.contains("media")) {
        db.createObjectStore("media", { keyPath: "id", autoIncrement: true });
      }
    };

    request.onsuccess = (e) => {
      db = e.target.result;
      statusEl.textContent = "";
      loadGallery();
    };

    photoInput.addEventListener("change", (e) => handlePhotoSelect(e));
    videoInput.addEventListener("change", (e) => handleVideoSelect(e));

    function handlePhotoSelect(e) {
      const file = e.target.files[0];
      if (!file) return;

      statusEl.textContent = "写真を保存中...";

      const img = new Image();
      const url = URL.createObjectURL(file);

      img.onload = () => {
        URL.revokeObjectURL(url);
        
        const maxSide = 1000;
        let width = img.width;
        let height = img.height;

        if (width > height && width > maxSide) {
          height = Math.round((height * maxSide) / width);
          width = maxSide;
        } else if (height > maxSide) {
          width = Math.round((width * maxSide) / height);
          height = maxSide;
        }

        const canvas = document.createElement("canvas");
        canvas.width = width;
        canvas.height = height;
        const ctx = canvas.getContext("2d");
        ctx.drawImage(img, 0, 0, width, height);

        const compressedData = canvas.toDataURL("image/jpeg", 0.7);
        saveToDB("image", compressedData, e.target);
      };

      img.onerror = () => {
        statusEl.textContent = "画像の読み込みに失敗しました。";
      };

      img.src = url;
    }

    function handleVideoSelect(e) {
      const file = e.target.files[0];
      if (!file) return;

      statusEl.textContent = "動画を保存中...";
      saveToDB("video", file, e.target);
    }

    function saveToDB(type, data, inputEl) {
      if (!db) {
        statusEl.textContent = "DBが開かれていません。";
        return;
      }

      try {
        const tx = db.transaction(["media"], "readwrite");
        const store = tx.objectStore("media");
        
        store.add({
          type: type,
          data: data,
          createdAt: new Date().getTime()
        });

        tx.oncomplete = () => {
          statusEl.textContent = "";
          loadGallery();
          inputEl.value = "";
        };

        tx.onerror = () => {
          statusEl.textContent = "保存中にエラーが発生しました。";
        };
      } catch (err) {
        statusEl.textContent = "エラー: " + err.message;
      }
    }

    function loadGallery() {
      if (!db) return;

      const tx = db.transaction(["media"], "readonly");
      const store = tx.objectStore("media");
      const req = store.getAll();

      req.onsuccess = () => {
        const items = req.result || [];
        galleryEl.innerHTML = "";

        if (items.length === 0) {
          statusEl.textContent = "保存されたデータはありません。";
          return;
        } else {
          statusEl.textContent = "";
        }

        items.sort((a, b) => b.id - a.id);

        items.forEach((item) => {
          const card = document.createElement("div");
          card.className = "media-card";

          if (item.type === "image") {
            const img = document.createElement("img");
            img.src = item.data;
            card.appendChild(img);
          } else {
            const video = document.createElement("video");
            if (item.data instanceof Blob) {
              video.src = URL.createObjectURL(item.data);
            } else {
              video.src = item.data;
            }
            video.controls = true;
            video.playsInline = true;
            card.appendChild(video);
          }

          const delBtn = document.createElement("button");
          delBtn.className = "delete-btn";
          delBtn.innerHTML = "✕";
          delBtn.onclick = () => deleteMedia(item.id);
          card.appendChild(delBtn);

          galleryEl.appendChild(card);
        });
      };

      req.onerror = () => {
        statusEl.textContent = "データの読み込みに失敗しました。";
      };
    }

    function deleteMedia(id) {
      if (!confirm("削除しますか？")) return;

      const tx = db.transaction(["media"], "readwrite");
      const store = tx.objectStore("media");
      store.delete(id);
      tx.oncomplete = () => loadGallery();
    }

    function resetDB() {
      if (confirm("すべての保存データを削除して初期化しますか？")) {
        indexedDB.deleteDatabase("SecretMediaDB");
        indexedDB.deleteDatabase("SecretMediaDB_v2");
        location.reload();
      }
    }
  </script>
</body>
</html>
