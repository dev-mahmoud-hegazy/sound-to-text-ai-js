/**
 * ============================================================================
 * SMART TTS MANAGER (Pure Core JS - Blazor Compatible)
 * Arabic Priority: Saudi (ar-SA) -> Egyptian (ar-EG) -> Any Arabic (ar)
 * English Priority: Natural/Google English -> US/UK English -> Default
 * Fallback URI: ms-settings:speech (Opens Windows Speech Settings if missing)
 * ============================================================================
 */
const TTSManager = {
  // 1. CONFIGURATION
  config: {
    windowsSettingsUri: "ms-settings:speech",
    rate: 1.0,
    pitch: 1.0,
    volume: 1.0,
    logVoicesOnLoad: true
  },

  state: {
    isInitialized: false
  },

  // 2. INITIALIZATION & CONSOLE LOGGING
  init: function () {
    if (this.state.isInitialized) return;
    this.state.isInitialized = true;

    const loadAndLog = () => {
      const voices = this.getAvailableVoices();
      if (voices.length > 0 && this.config.logVoicesOnLoad) {
        console.log("[TTSManager] Loaded system voices:", voices.length);
        console.table(
          voices.map((v, index) => ({
            index,
            name: v.name,
            lang: v.lang,
            local: v.localService
          }))
        );
      }
    };

    loadAndLog();
    if (window.speechSynthesis && window.speechSynthesis.onvoiceschanged !== undefined) {
      window.speechSynthesis.onvoiceschanged = loadAndLog;
    }
  },

  // 3. VOICE GETTERS & CHECKERS (Use later for UI dropdowns or validation)
  getAvailableVoices: function () {
    return window.speechSynthesis ? window.speechSynthesis.getVoices() : [];
  },

  getArabicVoices: function () {
    return this.getAvailableVoices().filter(v => v.lang.toLowerCase().startsWith("ar"));
  },

  getEnglishVoices: function () {
    return this.getAvailableVoices().filter(v => v.lang.toLowerCase().startsWith("en"));
  },

  hasArabicSupport: function () {
    return this.getArabicVoices().length > 0;
  },

  /**
   * Selects the best available voice based on text language and dialect priority.
   */
  selectBestVoice: function (isArabic, customVoiceName = null) {
    const voices = this.getAvailableVoices();
    if (!voices.length) return null;

    // Allow manual override if a specific voice name is passed
    if (customVoiceName) {
      const manual = voices.find(v => v.name === customVoiceName);
      if (manual) return manual;
    }

    if (isArabic) {
      // Priority 1: Saudi Arabic (Prefer Natural/Online Edge voices first)
      const saudi =
        voices.find(v => v.lang.toLowerCase().includes("ar-sa") && v.name.includes("Natural")) ||
        voices.find(v => v.lang.toLowerCase().includes("ar-sa"));
      if (saudi) return saudi;

      // Priority 2: Egyptian Arabic
      const egyptian =
        voices.find(v => v.lang.toLowerCase().includes("ar-eg") && v.name.includes("Natural")) ||
        voices.find(v => v.lang.toLowerCase().includes("ar-eg"));
      if (egyptian) return egyptian;

      // Priority 3: Any clear Arabic voice (UAE, Jordan, Standard Arabic, etc.)
      return voices.find(v => v.lang.toLowerCase().startsWith("ar")) || null;
    } else {
      // English Priority: Natural/Google -> US English -> Any English -> First available
      return (
        voices.find(v => v.lang.startsWith("en") && (v.name.includes("Natural") || v.name.includes("Google"))) ||
        voices.find(v => v.lang.toLowerCase().includes("en-us")) ||
        voices.find(v => v.lang.toLowerCase().startsWith("en")) ||
        voices[0] ||
        null
      );
    }
  },

  // 4. MAIN SPEAK FUNCTION (Entry point for Blazor & JS)
  /**
   * Speaks text and returns a structured result for Blazor UI handling.
   * @param {string} text - Arabic or English text
   * @param {object} options - Optional: { voiceName, rate, pitch, volume }
   * @returns {Promise<object>} { success, voice, lang, windowsSettingsUri, error }
   */
  speak: function (text, options = {}) {
    this.init();
    const cleanText = (text || "").trim();

    if (!cleanText) {
      return Promise.resolve({ success: false, error: "Empty text" });
    }

    if (!window.speechSynthesis) {
      return Promise.resolve({
        success: false,
        error: "Browser does not support speechSynthesis",
        windowsSettingsUri: this.config.windowsSettingsUri
      });
    }

    this.stop(); // Cancel any active speech before starting new

    const isArabic = /[\u0600-\u06FF]/.test(cleanText);
    const selectedVoice = this.selectBestVoice(isArabic, options.voiceName);

    // If no matching voice exists on Windows, return the Settings URI immediately
    if (!selectedVoice) {
      console.warn("[TTSManager] No suitable voice installed for:", isArabic ? "Arabic" : "English");
      return Promise.resolve({
        success: false,
        error: isArabic ? "No Arabic voice installed on Windows." : "No English voice found.",
        windowsSettingsUri: this.config.windowsSettingsUri
      });
    }

    return new Promise((resolve) => {
      const utterance = new SpeechSynthesisUtterance(cleanText);
      utterance.voice = selectedVoice;
      utterance.rate = options.rate || this.config.rate;
      utterance.pitch = options.pitch || this.config.pitch;
      utterance.volume = options.volume || this.config.volume;

      utterance.onend = () => {
        resolve({
          success: true,
          voice: selectedVoice.name,
          lang: selectedVoice.lang,
          windowsSettingsUri: null
        });
      };

      utterance.onerror = (event) => {
        console.error("[TTSManager] Playback error:", event.error);
        resolve({
          success: false,
          error: event.error,
          windowsSettingsUri: this.config.windowsSettingsUri
        });
      };

      window.speechSynthesis.speak(utterance);
    });
  },

  // 5. PLAYBACK CONTROLS
  stop: function () {
    if (window.speechSynthesis) {
      window.speechSynthesis.cancel();
    }
  }
};

// Auto-initialize so system voices load and log early
TTSManager.init();
