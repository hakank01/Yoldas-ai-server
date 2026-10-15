# Yoldas-ai-server
Package.json
<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta name="theme-color" content="#000000">
<title>Yoldaş</title>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body {
  width: 100%;
  height: 100%;
  overflow: hidden;
  background: #000;
  color: white;
  font-family: Arial, sans-serif;
}

#camera {
  position: fixed;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  background: #000;
}

#overlay {
  position: fixed;
  inset: 0;
  pointer-events: none;
  background: linear-gradient(
    to bottom,
    rgba(0,0,0,.55),
    transparent 30%,
    transparent 65%,
    rgba(0,0,0,.7)
  );
}

#status {
  position: fixed;
  top: 25px;
  left: 15px;
  right: 15px;
  text-align: center;
  font-size: 16px;
  text-shadow: 0 2px 5px black;
}

#message {
  position: fixed;
  bottom: 35px;
  left: 20px;
  right: 20px;
  text-align: center;
  font-size: 18px;
  line-height: 1.5;
  text-shadow: 0 2px 5px black;
}

#start {
  position: fixed;
  inset: 0;
  z-index: 10;
  background: #050505;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 28px;
  text-align: center;
}

#start h1 {
  font-size: 44px;
  letter-spacing: 4px;
  margin-bottom: 18px;
}

#start p {
  line-height: 1.7;
  color: #ddd;
  margin-bottom: 25px;
}

#start button {
  background: white;
  color: black;
  padding: 18px 25px;
  border: none;
  border-radius: 12px;
  font-size: 18px;
  font-weight: bold;
}

#start small {
  color: #aaa;
  margin-top: 20px;
  line-height: 1.5;
}
</style>
</head>

<body>

<video id="camera" autoplay muted playsinline></video>
<div id="overlay"></div>

<div id="status">YOLDAŞ • Başlamaya hazır</div>
<div id="message">Sesli yardımcın.</div>

<div id="start">
  <h1>YOLDAŞ</h1>
  <p>
    Merhaba! Ben Yoldaş.<br>
    Sana sesli olarak yardımcı olmak için buradayım.<br>
    Başlamak için aşağıdaki düğmeye dokun.
  </p>
  <button id="startButton">YOLDAŞ'I BAŞLAT</button>
  <small>
    Başladıktan sonra konuşarak komut verebilirsin.
  </small>
</div>

<script>
"use strict";

const camera = document.getElementById("camera");
const statusBox = document.getElementById("status");
const messageBox = document.getElementById("message");
const startScreen = document.getElementById("start");
const startButton = document.getElementById("startButton");

let started = false;
let speaking = false;
let listening = false;
let recognition = null;
let stream = null;
let language = "tr-TR";
let restartTimer = null;

/* Durum göster */
function status(text) {
  statusBox.textContent = "YOLDAŞ • " + text;
}

/* Konuşma metnini ekranda göster */
function show(text) {
  messageBox.textContent = text;
}

/*
  SES SİSTEMİ
  Türkçe sesleri bulmayı dener.
  Konuşma başlamazsa hata bilgisini ekrana yazar.
*/
function say(text, lang = language) {
  return new Promise((resolve) => {
    show(text);

    if (!("speechSynthesis" in window) ||
        !("SpeechSynthesisUtterance" in window)) {
      status("Seslendirme desteklenmiyor");
      resolve(false);
      return;
    }

    try {
      window.speechSynthesis.cancel();

      const utterance = new SpeechSynthesisUtterance(text);
      utterance.lang = lang;
      utterance.rate = 0.9;
      utterance.pitch = 1;
      utterance.volume = 1;

      const voices = window.speechSynthesis.getVoices();

      const preferred = voices.find(v =>
        v.lang.toLowerCase() === lang.toLowerCase()
      );

      const sameLanguage = voices.find(v =>
        v.lang.toLowerCase().startsWith(
          lang.substring(0, 2).toLowerCase()
        )
      );

      if (preferred || sameLanguage) {
        utterance.voice = preferred || sameLanguage;
      }

      speaking = true;
      status("Konuşuyor");

      utterance.onend = () => {
        speaking = false;
        status(started ? "Hazır" : "Başlamaya hazır");
        resolve(true);
      };

      utterance.onerror = (event) => {
        speaking = false;
        status("Ses hatası: " + event.error);
        resolve(false);
      };

      window.speechSynthesis.speak(utterance);

      /* Bazı Android WebView sürümlerinde konuşma kuyruğu takılabilir. */
      setTimeout(() => {
        if (speaking && window.speechSynthesis.paused) {
          window.speechSynthesis.resume();
        }
      }, 500);

    } catch (error) {
      speaking = false;
      status("Ses başlatılamadı");
      resolve(false);
    }
  });
}

/* Ses motorunun ses listesini hazırla */
if ("speechSynthesis" in window) {
  window.speechSynthesis.getVoices();

  window.speechSynthesis.onvoiceschanged = () => {
    window.speechSynthesis.getVoices();
  }
}

/* Kamerayı aç */
async function openCamera() {
  if (!navigator.mediaDevices ||
      !navigator.mediaDevices.getUserMedia) {
    throw new Error("Bu uygulamada kamera erişimi desteklenmiyor.");
  }

  stream = await navigator.mediaDevices.getUserMedia({
    audio: false,
    video: {
      facingMode: { ideal: "environment" }
    }
  });

  camera.srcObject = stream;
  await camera.play();
}

/* Mikrofon ve konuşma tanıma */
function startListening() {
  const SpeechAPI =
    window.SpeechRecognition ||
    window.webkitSpeechRecognition;

  if (!SpeechAPI) {
    status("Ses tanıma desteklenmiyor");
    say(
      "Sesli komut özelliği bu uygulama ortamında desteklenmiyor. HTML to APK uygulamasının mikrofon ve ses tanıma desteği gerekiyor."
    );
    return;
  }

  if (recognition) {
    try {
      recognition.stop();
    } catch (e) {}
  }

  recognition = new SpeechAPI();
  recognition.lang = language;
  recognition.continuous = true;
  recognition.interimResults = false;
  recognition.maxAlternatives = 1;

  recognition.onstart = () => {
    listening = true;
    status("Seni dinliyorum");
  };

  recognition.onresult = (event) => {
    const index = event.results.length - 1;

    if (!event.results[index].isFinal) return;

    const text = event.results[index][0].transcript.trim();

    if (text) {
      handleCommand(text);
    }
  };

  recognition.onerror = (event) => {
    listening = false;
    status("Mikrofon: " + event.error);
  };

  recognition.onend = () => {
    listening = false;

    if (started && !speaking) {
      clearTimeout(restartTimer);

      restartTimer = setTimeout(() => {
        try {
          recognition.start();
        } catch (e) {}
      }, 1000);
    }
  };

  try {
    recognition.start();
  } catch (error) {
    status("Mikrofon başlatılamadı");
  }
}

/* Konuşurken mikrofonun sesi kesmesini önlemeyi dene */
async function answer(text, lang = language) {
  if (recognition) {
    try {
      recognition.stop();
    } catch (e) {}
  }

  await say(text, lang);

  if (started && recognition) {
    setTimeout(() => {
      try {
        recognition.start();
      } catch (e) {}
    }, 400);
  }
}

/* Saat */
async function tellTime() {
  const time = new Date().toLocaleTimeString(language, {
    hour: "2-digit",
    minute: "2-digit"
  });

  await answer("Saat " + time);
}

/* Tarih */
async function tellDate() {
  const date = new Date().toLocaleDateString(language, {
    day: "numeric",
    month: "long",
    year: "numeric"
  });

  await answer("Bugünün tarihi " + date);
}

/* Konumu al */
function getPosition() {
  return new Promise((resolve, reject) => {
    if (!navigator.geolocation) {
      reject(new Error("Konum desteklenmiyor."));
      return;
    }

    navigator.geolocation.getCurrentPosition(
      resolve,
      reject,
      {
        enableHighAccuracy: true,
        timeout: 15000,
        maximumAge: 0
      }
    );
  });
}

/* Gideceğin yeri sorup haritaya aktar */
async function navigate() {
  await answer(
    language === "tr-TR"
      ? "Nereye gitmek istediğini söyleyebilirsin. Örneğin, en yakın markete git."
      : "Tell me where you want to go. For example, take me to the nearest market."
  );

  /*
    Kullanıcının söylediği hedef, ayrı bir komut olarak
    handleCommand içinde algılanır.
  */
}

/* Hedefi haritada aç */
function openMap(destination) {
  const url =
    "https://www.google.com/maps/dir/?api=1&destination=" +
    encodeURIComponent(destination) +
    "&travelmode=walking";

  window.open(url, "_blank");
}

/* Kameradan fotoğraf al */
function takePhoto() {
  if (!camera.videoWidth || !camera.videoHeight) {
    throw new Error("Kamera görüntüsü hazır değil.");
  }

  const canvas = document.createElement("canvas");
  canvas.width = camera.videoWidth;
  canvas.height = camera.videoHeight;

  canvas.getContext("2d").drawImage(
    camera, 0, 0, canvas.width, canvas.height
  );

  return canvas.toDataURL("image/jpeg", 0.8);
}

/*
  Bu özellikler sunucuya ihtiyaç duyar.
  /api/vision ve /api/ocr örnek adreslerdir.
*/
async function analyzeScene() {
  await answer("Çevreni analiz etmeye çalışıyorum.");

  try {
    const image = takePhoto();

    const response = await fetch("/api/vision", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ image, language })
    });

    if (!response.ok) throw new Error("Sunucu hatası");

    const data = await response.json();

    await answer(
      data.description || "Görüntü açıklaması alınamadı."
    );
  } catch (error) {
    await answer(
      "Çevreyi anlatmak için yapay zekâ sunucusunun kurulması gerekiyor."
    );
  }
}

async function readText() {
  await answer("Görüntüdeki yazıyı okumaya çalışıyorum.");

  try {
    const image = takePhoto();

    const response = await fetch("/api/ocr", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ image, language })
    });

    if (!response.ok) throw new Error("Sunucu hatası");

    const data = await response.json();

    await answer(
      data.text || "Okunabilir bir metin bulunamadı."
    );
  } catch (error) {
    await answer(
      "Yazı okuma özelliği için OCR sunucusunun kurulması gerekiyor."
    );
  }
}

/* Türkçe ve İngilizce komutlar */
async function handleCommand(raw) {
  if (speaking) return;

  const text = raw.toLocaleLowerCase(
    language === "tr-TR" ? "tr-TR" : "en-US"
  );

  status("Komut alındı");
  show("Duyduğum: " + raw);

  if (language === "tr-TR") {
    if (text.includes("çevremi anlat") ||
        text.includes("etrafımı anlat") ||
        text.includes("ne görüyorsun")) {
      await analyzeScene();

    } else if (text.includes("yazıyı oku") ||
               text.includes("metni oku")) {
      await readText();

    } else if (text.includes("saat kaç")) {
      await tellTime();

    } else if (text.includes("bugünün tarihi") ||
               text.includes("tarih ne")) {
      await tellDate();

    } else if (text.includes("ingilizceye geç")) {
      language = "en-US";
      await answer("English mode enabled.", "en-US");

    } else if (text.includes("türkçeye geç")) {
      language = "tr-TR";
      await answer("Türkçe modu açıldı.", "tr-TR");

    } else if (text.includes("yardım") ||
               text.includes("komutlar")) {
      await answer(
        "Şunları söyleyebilirsin: Yoldaş, çevremi anlat, yazıyı oku, saat kaç, bugünün tarihi, İngilizceye geç veya yardım."
      );

    } else if (text.includes("yoldaş")) {
      await answer("Buradayım, seni dinliyorum.");

    } else if (text.startsWith("beni ")) {
      const destination = raw.substring(5).trim();
      if (destination) {
        openMap(destination);
        await answer("Haritayı açıyorum. Yol tarifini kontrol et.");
      } else {
        await answer("Gitmek istediğin yeri tekrar söyle.");
      }

    } else if (text.startsWith("git ")) {
      const destination = raw.substring(4).trim();
      if (destination) {
        openMap(destination);
        await answer("Haritayı açıyorum. Yol tarifini kontrol et.");
      } else {
        await answer("Gitmek istediğin yeri söyle.");
      }

    } else if (text.includes("navigasyonu aç")) {
      await navigate();

    } else {
      await answer(
        "Komutunu anlayamadım. Yardım diyerek komutları öğrenebilirsin."
      );
    }

  } else {
    if (text.includes("describe my surroundings")) {
      await analyzeScene();

    } else if (text.includes("read the text")) {
      await readText();

    } else if (text.includes("what time is it")) {
      await tellTime();

    } else if (text.includes("what is today's date")) {
      await tellDate();

    } else if (text.includes("switch to turkish")) {
      language = "tr-TR";
      await answer("Türkçe modu açıldı.", "tr-TR");

    } else if (text.includes("help")) {
      await answer(
        "Say describe my surroundings, read the text, what time is it, or switch to Turkish.",
        "en-US"
      );

    } else {
      await answer(
        "I did not understand. Say help for available commands.",
        "en-US"
      );
    }
  }
}

/* Başlat düğmesi */
startButton.addEventListener("click", async () => {
  if (started) return;

  startButton.disabled = true;
  startButton.textContent = "BAŞLATILIYOR...";

  /*
    İlk ses doğrudan kullanıcı dokunuşuyla başlatılır.
    Android tarayıcılarının ses iznini kabul etmesi için
    bu sıra önemlidir.
  */
  const firstSpeech = say(
    "Merhaba! Ben Yoldaş. Seni dinlemeye hazırım."
  );

  started = true;
  startScreen.style.display = "none";

  try {
    await openCamera();
    status("Kamera açıldı");
  } catch (error) {
    status("Kamera açılamadı");
    show("Kamera iznini kontrol et. Sesli yardımcının bazı özellikleri yine çalışabilir.");
  }

  await firstSpeech;

  startListening();

  if (!recognition) {
    await say("Mikrofon desteği kullanılamıyor.");
  }
});
</script>

</body>
</html>

import express from "express";
import cors from "cors";
import OpenAI from "openai";
import rateLimit from "express-rate-limit";

const app = express();
const port = process.env.PORT || 3000;

app.use(cors());
app.use(express.json({ limit: "10mb" }));

// Her IP için istek sınırı
app.use("/api/", rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 20,
  standardHeaders: true,
  legacyHeaders: false
}));

const apiKey = process.env.OPENAI_API_KEY;

const ai = apiKey
  ? new OpenAI({ apiKey })
  : null;

app.get("/", (req, res) => {
  res.json({
    app: "Yoldaş",
    status: "Sunucu çalışıyor"
  });
});

function getImage(image) {
  if (typeof image !== "string" ||
      !/^data:image\/(jpeg|png|webp);base64,/i.test(image)) {
    throw new Error("Geçerli bir JPEG, PNG veya WEBP görüntüsü gerekli.");
  }

  if (image.length > 9_000_000) {
    throw new Error("Görüntü çok büyük. Daha küçük bir fotoğraf gönder.");
  }

  return image;
}

async function askAI(image, prompt) {
  if (!ai) {
    throw new Error("OPENAI_API_KEY sunucu ayarına eklenmemiş.");
  }

  const result = await ai.responses.create({
    model: "gpt-4.1-mini",
    input: [{
      role: "user",
      content: [
        {
          type: "input_text",
          text: prompt
        },
        {
          type: "input_image",
          image_url: image,
          detail: "high"
        }
      ]
    }]
  });

  return result.output_text || "Bir cevap oluşturulamadı.";
}

// Çevre analizi
app.post("/api/vision", async (req, res) => {
  try {
    const image = getImage(req.body.image);
    const lang = req.body.language === "en-US"
      ? "English"
      : "Turkish";

    const description = await askAI(
      image,
      `You are Yoldas, a visual assistant for a blind
      or visually impaired person. Describe this image
      concisely in ${lang}. Mention clearly visible objects,
      signs, people, and possible obstacles. Distinguish
      uncertain details from facts. Do not claim that a
      path is safe or that the image shows the user's
      complete surroundings. Give a useful spoken answer.`
    );

    res.json({ description });
  } catch (error) {
    console.error("Vision error:", error.message);
    res.status(500).json({
      error: "Görüntü analiz edilemedi."
    });
  }
});

// Fotoğraftaki yazıları okuma (OCR)
app.post("/api/ocr", async (req, res) => {
  try {
    const image = getImage(req.body.image);
    const lang = req.body.language === "en-US"
      ? "English"
      : "Turkish";

    const text = await askAI(
      image,
      `Read all clearly visible text in this image.
      Return the text in its original language and
      preserve the reading order as well as possible.
      Do not invent missing or blurry words. If a word
      cannot be read, mark it as unclear. Reply in
      ${lang}, but do not translate the original text
      unless asked.`
    );

    res.json({ text });
  } catch (error) {
    console.error("OCR error:", error.message);
    res.status(500).json({
      error: "Yazılar okunamadı."
    });
  }
});

app.listen(port, "0.0.0.0", () => {
  console.log(`Yoldaş sunucusu ${port} portunda çalışıyor.`);
});
