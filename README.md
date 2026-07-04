# CodeA1pha_Language-Translation-Tool
CodeAlpha Language Translation Tool is a simple application that translates text from one language to another using translation APIs or AI models. Users select the source and target languages enter text and get instant accurate translations. It helps improve communication supports language learning and enables multilingual interaction efficiently.

Author- Prem Somnath Sonawane

TASK 1:
Language Translation Tool
             Create a user interface where user can enter text and
select source & target languages.  Use a translation API like Google Translate
API or Microsoft Translator to process the input.
             Send
the text to the API and get the translated response.
             Display
the translated text clearly on the screen.
             Optional:
Add a copy button or text-to-speech feature for better usability.



*CODE Execution:

import React, { useState, useEffect } from "react";
import { 
  Languages, ArrowLeftRight, Copy, Volume2, Trash2, 
  History, X, Check, Loader2, Sparkles, AlertCircle,
  Clock, Keyboard, Info, CheckCircle2, RefreshCw, HelpCircle,
  ExternalLink, VolumeX, ChevronDown, BookOpen, Smartphone, Activity
} from "lucide-react";
import { motion, AnimatePresence } from "motion/react";

// List of supported major languages
const LANGUAGES = [
  { code: "auto", name: "Auto-detect" },
  { code: "en", name: "English" },
  { code: "es", name: "Spanish" },
  { code: "fr", name: "French" },
  { code: "de", name: "German" },
  { code: "zh", name: "Chinese (Simplified)" },
  { code: "hi", name: "Hindi" },
  { code: "ar", name: "Arabic" },
  { code: "ja", name: "Japanese" },
  { code: "it", name: "Italian" },
  { code: "pt", name: "Portuguese" },
  { code: "ru", name: "Russian" },
  { code: "ko", name: "Korean" },
  { code: "tr", name: "Turkish" },
  { code: "nl", name: "Dutch" },
  { code: "vi", name: "Vietnamese" },
  { code: "sv", name: "Swedish" }
];

// Target languages (excludes Auto-detect)
const TARGET_LANGUAGES = LANGUAGES.filter(l => l.code !== "auto");

// Map standard language codes to speech synthesis locales
const SPEECH_LANGS: Record<string, string> = {
  en: "en-US",
  es: "es-ES",
  fr: "fr-FR",
  de: "de-DE",
  zh: "zh-CN",
  hi: "hi-IN",
  ar: "ar-SA",
  ja: "ja-JP",
  it: "it-IT",
  pt: "pt-BR",
  ru: "ru-RU",
  ko: "ko-KR",
  tr: "tr-TR",
  nl: "nl-NL",
  vi: "vi-VN",
  sv: "sv-SE"
};

interface HistoryItem {
  id: string;
  sourceText: string;
  translatedText: string;
  sourceLang: string;
  targetLang: string;
  engine: string;
  timestamp: string;
}

export default function App() {
  const [sourceText, setSourceText] = useState("");
  const [translatedText, setTranslatedText] = useState("");
  const [sourceLang, setSourceLang] = useState("auto");
  const [targetLang, setTargetLang] = useState("es");
  const [detectedLangCode, setDetectedLangCode] = useState("");
  const [detectedLangName, setDetectedLangName] = useState("");
  
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [engine, setEngine] = useState<"gemini" | "mymemory">("gemini");
  const [autoTranslate, setAutoTranslate] = useState(true);
  
  const [history, setHistory] = useState<HistoryItem[]>([]);
  const [showInfo, setShowInfo] = useState(true);
  
  const [copySuccess, setCopySuccess] = useState(false);
  const [isSpeakingSource, setIsSpeakingSource] = useState(false);
  const [isSpeakingTarget, setIsSpeakingTarget] = useState(false);

  // Load history from localStorage on mount
  useEffect(() => {
    try {
      const storedHistory = localStorage.getItem("translation_history");
      if (storedHistory) {
        setHistory(JSON.parse(storedHistory));
      }
    } catch (e) {
      console.error("Failed to load history from localStorage", e);
    }
  }, []);

  // Cleanup SpeechSynthesis on unmount
  useEffect(() => {
    return () => {
      if (window.speechSynthesis) {
        window.speechSynthesis.cancel();
      }
    };
  }, []);

  // Main Translation Handler
  const handleTranslate = async (force: boolean = false) => {
    const textToTranslate = sourceText.trim();
    if (!textToTranslate) {
      setTranslatedText("");
      setDetectedLangCode("");
      setDetectedLangName("");
      return;
    }

    if (!force && !autoTranslate) return;

    setIsLoading(true);
    setError(null);

    try {
      const response = await fetch("/api/translate", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          text: textToTranslate,
          sourceLang,
          targetLang,
          engine
        })
      });

      if (!response.ok) {
        const errData = await response.json().catch(() => ({}));
        throw new Error(errData.error || `Translation API returned status ${response.status}`);
      }

      const data = await response.json();
      setTranslatedText(data.translatedText);

      if (sourceLang === "auto" && data.detectedSourceLang) {
        setDetectedLangCode(data.detectedSourceLang);
        setDetectedLangName(data.detectedLanguageName);
      } else {
        setDetectedLangCode("");
        setDetectedLangName("");
      }

      // Add to history state and local storage
      const newHistoryItem: HistoryItem = {
        id: Date.now().toString(),
        sourceText: textToTranslate,
        translatedText: data.translatedText,
        sourceLang: sourceLang === "auto" ? data.detectedSourceLang : sourceLang,
        targetLang,
        engine: data.engine || engine,
        timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
      };

      setHistory(prev => {
        // Prevent exact duplicates in quick history
        const filtered = prev.filter(item => item.sourceText !== textToTranslate);
        const updated = [newHistoryItem, ...filtered].slice(0, 30);
        localStorage.setItem("translation_history", JSON.stringify(updated));
        return updated;
      });

    } catch (err: any) {
      console.error("Translation request error:", err);
      setError(err.message || "Failed to contact translator server.");
    } finally {
      setIsLoading(false);
    }
  };

  // Debounced translation for 'instant as-you-type' translations
  useEffect(() => {
    if (!autoTranslate) return;
    if (!sourceText.trim()) {
      setTranslatedText("");
      setDetectedLangCode("");
      setDetectedLangName("");
      return;
    }

    const timer = setTimeout(() => {
      handleTranslate();
    }, 600); // 600ms debounce typing gap

    return () => clearTimeout(timer);
  }, [sourceText, sourceLang, targetLang, autoTranslate, engine]);

  // Manually force translate
  const triggerManualTranslate = () => {
    handleTranslate(true);
  };

  // Swap Languages
  const handleSwap = () => {
    const activeSource = sourceLang === "auto" ? (detectedLangCode || "en") : sourceLang;
    const activeTarget = targetLang;

    // Swap text values
    const tempText = sourceText;
    setSourceText(translatedText);
    setTranslatedText(tempText);

    // Swap dropdown selection codes
    setSourceLang(activeTarget);
    setTargetLang(activeSource);
  };

  // Copy text to clipboard with animation state
  const copyToClipboard = (text: string) => {
    if (!text) return;
    navigator.clipboard.writeText(text);
    setCopySuccess(true);
    setTimeout(() => setCopySuccess(false), 2000);
  };

  // Native Speech Synthesis
  const speakText = (text: string, langCode: string, isSource: boolean) => {
    if (!window.speechSynthesis) {
      alert("Text-to-speech is not supported in this browser.");
      return;
    }

    window.speechSynthesis.cancel();

    // Toggle off if clicking while currently speaking
    if (isSource && isSpeakingSource) {
      setIsSpeakingSource(false);
      return;
    }
    if (!isSource && isSpeakingTarget) {
      setIsSpeakingTarget(false);
      return;
    }

    const utterance = new SpeechSynthesisUtterance(text);
    const resolvedLang = SPEECH_LANGS[langCode] || langCode;
    utterance.lang = resolvedLang;

    utterance.onend = () => {
      if (isSource) setIsSpeakingSource(false);
      else setIsSpeakingTarget(false);
    };

    utterance.onerror = () => {
      if (isSource) setIsSpeakingSource(false);
      else setIsSpeakingTarget(false);
    };

    if (isSource) {
      setIsSpeakingSource(true);
      setIsSpeakingTarget(false);
    } else {
      setIsSpeakingTarget(true);
      setIsSpeakingSource(false);
    }

    window.speechSynthesis.speak(utterance);
  };

  // Delete individual history item
  const deleteHistoryItem = (id: string, e: React.MouseEvent) => {
    e.stopPropagation();
    setHistory(prev => {
      const updated = prev.filter(item => item.id !== id);
      localStorage.setItem("translation_history", JSON.stringify(updated));
      return updated;
    });
  };

  // Clear all history
  const clearAllHistory = () => {
    setHistory([]);
    localStorage.removeItem("translation_history");
  };

  // Load selected item from history back into the translator
  const loadHistoryItem = (item: HistoryItem) => {
    setSourceText(item.sourceText);
    setTranslatedText(item.translatedText);
    setSourceLang(item.sourceLang);
    setTargetLang(item.targetLang);
    setEngine(item.engine as "gemini" | "mymemory");
  };

  return (
    <div className="flex flex-col min-h-screen bg-slate-950 text-slate-200 font-sans overflow-x-hidden" id="app_root">
      
      {/* Top Navigation Bar */}
      <nav className="h-16 border-b border-slate-800/80 flex items-center justify-between px-6 sm:px-8 bg-slate-950/70 backdrop-blur-md sticky top-0 z-50">
        <div className="flex items-center gap-3">
          <div className="w-8 h-8 bg-blue-600 rounded-lg flex items-center justify-center shadow-lg shadow-blue-500/20">
            <Languages className="w-5 h-5 text-white" />
          </div>
          <span className="font-extrabold text-lg sm:text-xl tracking-tight text-white font-display">
            AuraTranslate<span className="text-blue-500">.</span>
          </span>
        </div>
        
        {/* Status Indicators in Header */}
        <div className="hidden sm:flex items-center gap-4 text-xs font-medium text-slate-400">
          <div className="flex items-center gap-1.5 px-2.5 py-1 rounded-full bg-slate-900 border border-slate-800">
            <Sparkles className="w-3.5 h-3.5 text-indigo-400" />
            <span className="text-[11px] text-slate-300">Gemini 3.5 Active</span>
          </div>
          <div className="flex items-center gap-1.5 px-2.5 py-1 rounded-full bg-slate-900 border border-slate-800">
            <span className="w-1.5 h-1.5 bg-green-500 rounded-full animate-pulse" />
            <span className="text-[11px] text-slate-300">API Live</span>
          </div>
        </div>
      </nav>

      {/* Main Content Grid (Bento Layout) */}
      <main className="flex-1 p-4 sm:p-6 lg:p-8 max-w-7xl w-full mx-auto grid grid-cols-12 gap-5 z-10" id="bento_container">
        
        {/* BENTO BLOCK 1: Main Translator (col-span-12 lg:col-span-8) */}
        <section className="col-span-12 lg:col-span-8 bg-slate-900 border border-slate-800 rounded-2xl shadow-2xl flex flex-col overflow-hidden" id="main_translator_bento">
          
          {/* Top Controls: Dropdowns */}
          <div className="flex flex-col sm:flex-row sm:items-center justify-between px-5 sm:px-6 py-4 border-b border-slate-800/80 bg-slate-900/40 gap-3">
            <div className="flex items-center gap-3 flex-1">
              
              {/* Source Select */}
              <div className="relative">
                <select
                  id="source_lang_select"
                  value={sourceLang}
                  onChange={(e) => setSourceLang(e.target.value)}
                  className="bg-transparent text-sm font-semibold text-blue-400 outline-none cursor-pointer pr-6 appearance-none focus:text-blue-300 transition-colors"
                >
                  {LANGUAGES.map((lang) => (
                    <option key={`src-${lang.code}`} value={lang.code} className="bg-slate-900 text-slate-200">
                      {lang.name}
                    </option>
                  ))}
                </select>
                <ChevronDown className="w-3.5 h-3.5 text-blue-400 absolute right-0 top-1/2 -translate-y-1/2 pointer-events-none" />
              </div>

              {/* Intersecting Swap Action */}
              <button
                type="button"
                id="swap_langs_btn"
                onClick={handleSwap}
                className="p-2 hover:bg-slate-800 rounded-lg text-slate-500 hover:text-slate-200 transition-all active:scale-95 duration-150"
                title="Swap Languages"
              >
                <ArrowLeftRight className="w-4 h-4 text-blue-500" />
              </button>

              {/* Target Select */}
              <div className="relative">
                <select
                  id="target_lang_select"
                  value={targetLang}
                  onChange={(e) => setTargetLang(e.target.value)}
                  className="bg-transparent text-sm font-semibold text-slate-200 outline-none cursor-pointer pr-6 appearance-none focus:text-white transition-colors"
                >
                  {TARGET_LANGUAGES.map((lang) => (
                    <option key={`tgt-${lang.code}`} value={lang.code} className="bg-slate-900 text-slate-200">
                      {lang.name}
                    </option>
                  ))}
                </select>
                <ChevronDown className="w-3.5 h-3.5 text-slate-400 absolute right-0 top-1/2 -translate-y-1/2 pointer-events-none" />
              </div>
            </div>

            {/* Quick Engine Toggle and Status */}
            <div className="flex items-center gap-3 justify-between sm:justify-start">
              <div className="flex items-center bg-slate-950 p-1.5 rounded-xl border border-slate-800">
                <button
                  type="button"
                  onClick={() => setEngine("gemini")}
                  className={`px-3 py-1 text-xs font-semibold rounded-lg transition-all duration-150 flex items-center gap-1 ${
                    engine === "gemini"
                      ? "bg-blue-600 text-white shadow-md"
                      : "text-slate-400 hover:text-slate-200"
                  }`}
                >
                  <Sparkles className="w-3 h-3" />
                  Gemini
                </button>
                <button
                  type="button"
                  onClick={() => setEngine("mymemory")}
                  className={`px-3 py-1 text-xs font-semibold rounded-lg transition-all duration-150 ${
                    engine === "mymemory"
                      ? "bg-blue-600 text-white shadow-md"
                      : "text-slate-400 hover:text-slate-200"
                  }`}
                >
                  MyMemory
                </button>
              </div>
            </div>
          </div>

          {/* Editor Layout Pane */}
          <div className="flex-1 flex flex-col md:flex-row divide-y md:divide-y-0 md:divide-x divide-slate-800/80 bg-slate-950/20">
            
            {/* Input Column */}
            <div className="flex-1 flex flex-col p-6 min-h-[220px]">
              <div className="flex-1 relative flex flex-col">
                <textarea
                  id="source_textarea"
                  value={sourceText}
                  onChange={(e) => setSourceText(e.target.value)}
                  maxLength={5000}
                  placeholder="Enter text to translate..."
                  className="w-full flex-1 bg-transparent resize-none outline-none text-base sm:text-lg text-white placeholder-slate-600 leading-relaxed pr-6"
                />
                {sourceText && (
                  <button
                    type="button"
                    onClick={() => {
                      setSourceText("");
                      setTranslatedText("");
                      setDetectedLangCode("");
                      setDetectedLangName("");
                    }}
                    className="absolute right-0 top-0 p-1 rounded-full text-slate-500 hover:text-slate-300 hover:bg-slate-800/40 transition-colors"
                    title="Clear text"
                  >
                    <X className="w-4.5 h-4.5" />
                  </button>
                )}
              </div>

              {/* Input Action row */}
              <div className="flex justify-between items-center mt-4 pt-4 border-t border-slate-800/30">
                <div className="flex gap-1.5">
                  <button
                    type="button"
                    onClick={() => speakText(sourceText, sourceLang === "auto" ? (detectedLangCode || "en") : sourceLang, true)}
                    disabled={!sourceText}
                    className={`p-2 rounded-lg text-slate-400 hover:text-slate-200 transition-colors ${
                      isSpeakingSource ? "bg-blue-600/20 text-blue-400 ring-1 ring-blue-500/30" : "hover:bg-slate-800"
                    } disabled:opacity-30 disabled:cursor-not-allowed`}
                    title="Speak text"
                  >
                    {isSpeakingSource ? <VolumeX className="w-5 h-5" /> : <Volume2 className="w-5 h-5" />}
                  </button>
                  {isSpeakingSource && (
                    <span className="flex items-center gap-1.5 px-2 py-1 rounded bg-blue-950/50 text-[10px] text-blue-400 border border-blue-900/30 font-mono">
                      Speaking...
                    </span>
                  )}
                </div>
                <span className="text-[11px] text-slate-500 font-mono">{sourceText.length} / 5000</span>
              </div>
            </div>

            {/* Output Column */}
            <div className="flex-1 bg-slate-900/30 flex flex-col p-6 min-h-[220px]">
              <div className="flex-1 overflow-y-auto min-h-[140px] relative">
                {isLoading ? (
                  <div className="space-y-3 py-1 pr-4">
                    <div className="h-4 bg-slate-800/50 rounded animate-pulse w-full" />
                    <div className="h-4 bg-slate-800/50 rounded animate-pulse w-5/6" />
                    <div className="h-4 bg-slate-800/50 rounded animate-pulse w-3/4" />
                  </div>
                ) : error ? (
                  <div className="flex gap-2 text-rose-400 text-sm bg-rose-950/20 border border-rose-900/30 p-4 rounded-xl">
                    <AlertCircle className="w-5 h-5 shrink-0 mt-0.5" />
                    <div>
                      <p className="font-semibold">Translation failed</p>
                      <p className="text-xs text-rose-300/90 mt-0.5">{error}</p>
                      <button 
                        type="button" 
                        onClick={triggerManualTranslate}
                        className="text-xs text-blue-400 font-semibold hover:text-blue-300 mt-2 flex items-center gap-1 hover:underline"
                      >
                        <RefreshCw className="w-3 h-3" /> Retry translation
                      </button>
                    </div>
                  </div>
                ) : translatedText ? (
                  <p className="text-base sm:text-lg text-blue-400 whitespace-pre-wrap select-text leading-relaxed pr-2">
                    {translatedText}
                  </p>
                ) : (
                  <p className="text-slate-650 text-sm sm:text-base italic">
                    Translation will appear here...
                  </p>
                )}

                {/* Auto detected label overlay */}
                {detectedLangName && sourceLang === "auto" && !isLoading && !error && (
                  <div className="absolute top-0 right-0 bg-blue-950/60 border border-blue-900/50 px-2.5 py-1 rounded-full text-[10px] text-blue-300 font-medium shadow-inner flex items-center gap-1">
                    <CheckCircle2 className="w-3 h-3 text-blue-400" />
                    Detected: <span className="font-bold text-blue-200">{detectedLangName}</span>
                  </div>
                )}
              </div>

              {/* Output Action row */}
              <div className="flex justify-between items-center mt-4 pt-4 border-t border-slate-800/30">
                <div className="flex gap-1.5 items-center">
                  <button
                    type="button"
                    onClick={() => speakText(translatedText, targetLang, false)}
                    disabled={!translatedText || isLoading}
                    className={`p-2 rounded-lg text-slate-400 hover:text-slate-200 transition-colors ${
                      isSpeakingTarget ? "bg-blue-600/20 text-blue-400 ring-1 ring-blue-500/30" : "hover:bg-slate-800"
                    } disabled:opacity-30 disabled:cursor-not-allowed`}
                    title="Speak translation"
                  >
                    {isSpeakingTarget ? <VolumeX className="w-5 h-5" /> : <Volume2 className="w-5 h-5" />}
                  </button>

                  <button
                    type="button"
                    onClick={() => copyToClipboard(translatedText)}
                    disabled={!translatedText || isLoading}
                    className="p-2 hover:bg-slate-800 rounded-lg text-slate-400 hover:text-slate-200 transition-colors disabled:opacity-30 disabled:cursor-not-allowed relative"
                    title="Copy to Clipboard"
                  >
                    <AnimatePresence mode="wait">
                      {copySuccess ? (
                        <motion.span
                          key="copied-check"
                          initial={{ scale: 0.8, opacity: 0 }}
                          animate={{ scale: 1, opacity: 1 }}
                          exit={{ scale: 0.8, opacity: 0 }}
                          className="text-emerald-400"
                        >
                          <Check className="w-5 h-5" />
                        </motion.span>
                      ) : (
                        <Copy className="w-5 h-5" />
                      )}
                    </AnimatePresence>
                  </button>

                  {copySuccess && (
                    <span className="text-[10px] font-bold text-emerald-400 bg-emerald-500/10 px-2 py-0.5 rounded border border-emerald-500/20 font-mono animate-fade-in">
                      Copied!
                    </span>
                  )}

                  {isSpeakingTarget && (
                    <span className="flex items-center gap-1.5 px-2 py-1 rounded bg-blue-950/50 text-[10px] text-blue-400 border border-blue-900/30 font-mono">
                      Speaking...
                    </span>
                  )}
                </div>

                <div className="flex items-center gap-3">
                  {translatedText && !isLoading && (
                    <span className="text-[10px] font-semibold text-slate-500 font-mono">
                      Engine: {engine === "gemini" ? "Gemini AI" : "MyMemory"}
                    </span>
                  )}
                  <button
                    type="button"
                    onClick={triggerManualTranslate}
                    disabled={isLoading || !sourceText.trim()}
                    className="px-5 py-2 bg-blue-600 hover:bg-blue-500 disabled:bg-slate-800 disabled:opacity-50 text-white font-bold rounded-lg transition-all shadow-lg shadow-blue-900/20 text-xs sm:text-sm cursor-pointer hover:scale-102 active:scale-98"
                  >
                    {isLoading ? "Translating..." : "Translate"}
                  </button>
                </div>
              </div>
            </div>
          </div>

          {/* Bottom Settings Toggle bar */}
          <div className="px-5 py-3 border-t border-slate-800/80 bg-slate-950/30 flex flex-col sm:flex-row sm:items-center justify-between gap-3 text-xs text-slate-400">
            <label className="flex items-center gap-2.5 cursor-pointer group" htmlFor="auto_translate_toggle">
              <div className="relative">
                <input
                  type="checkbox"
                  id="auto_translate_toggle"
                  checked={autoTranslate}
                  onChange={(e) => setAutoTranslate(e.target.checked)}
                  className="sr-only peer"
                />
                <div className="w-8 h-4.5 bg-slate-800 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-slate-400 after:rounded-full after:h-3.5 after:w-3.5 after:transition-all peer-checked:bg-blue-600 peer-checked:after:bg-white" />
              </div>
              <span className="font-semibold text-slate-300 group-hover:text-white transition-colors">Instant Translation as you type</span>
            </label>
            <div className="flex gap-4 font-mono text-[11px] text-slate-500 justify-between sm:justify-start">
              <span>Layout: Split-pane Bento</span>
              <span>Input Limit: 5k</span>
            </div>
          </div>
        </section>

        {/* BENTO BLOCK 2: Recent History Sidebar (col-span-12 lg:col-span-4) */}
        <section className="col-span-12 lg:col-span-4 bg-slate-900 border border-slate-800 rounded-2xl p-6 flex flex-col justify-between shadow-2xl min-h-[400px]" id="history_bento">
          <div className="flex flex-col flex-1">
            <div className="flex items-center justify-between mb-4 pb-3 border-b border-slate-800/50">
              <h3 className="text-sm font-extrabold text-white uppercase tracking-wider flex items-center gap-1.5 font-display">
                <History className="w-4 h-4 text-blue-500" />
                Recent History
              </h3>
              {history.length > 0 && (
                <button
                  type="button"
                  onClick={clearAllHistory}
                  className="text-[11px] text-rose-400 hover:text-rose-300 transition-colors flex items-center gap-1 font-semibold"
                >
                  <Trash2 className="w-3.5 h-3.5" />
                  Clear All
                </button>
              )}
            </div>

            {/* List Body */}
            <div className="flex-1 space-y-3 max-h-[380px] overflow-y-auto pr-1">
              {history.length === 0 ? (
                <div className="h-full flex flex-col items-center justify-center text-center py-12 text-slate-500 px-4">
                  <Clock className="w-8 h-8 text-slate-600 mb-2" />
                  <p className="text-xs font-semibold">No recent translations</p>
                  <p className="text-[10px] text-slate-650 mt-1 max-w-[200px]">Your translated text snippets will automatically persist in your local history.</p>
                </div>
              ) : (
                <AnimatePresence initial={false}>
                  {history.map((item) => (
                    <motion.div
                      key={item.id}
                      initial={{ opacity: 0, y: 10 }}
                      animate={{ opacity: 1, y: 0 }}
                      exit={{ opacity: 0, x: -10 }}
                      onClick={() => loadHistoryItem(item)}
                      className="group relative p-3 bg-slate-850 hover:bg-slate-800 border border-slate-800/80 hover:border-slate-700 rounded-xl cursor-pointer transition-all duration-150 flex items-start justify-between gap-3"
                    >
                      <div className="flex-1 min-w-0">
                        <div className="flex items-center gap-2 mb-1">
                          <p className="text-[10px] font-bold text-blue-400 uppercase tracking-wider font-mono">
                            {LANGUAGES.find(l => l.code === item.sourceLang)?.name || item.sourceLang} &rarr; {LANGUAGES.find(l => l.code === item.targetLang)?.name || item.targetLang}
                          </p>
                          <span className="text-[9px] text-slate-500 font-mono">{item.timestamp}</span>
                        </div>
                        <p className="text-xs text-slate-300 font-medium truncate mb-0.5">{item.sourceText}</p>
                        <p className="text-xs text-blue-300 truncate font-mono">{item.translatedText}</p>
                      </div>
                      <button
                        type="button"
                        onClick={(e) => deleteHistoryItem(item.id, e)}
                        className="p-1 rounded text-slate-500 hover:text-rose-400 hover:bg-rose-500/10 opacity-0 group-hover:opacity-100 transition-all duration-150 shrink-0"
                        title="Delete from history"
                      >
                        <X className="w-3.5 h-3.5" />
                      </button>
                    </motion.div>
                  ))}
                </AnimatePresence>
              )}
            </div>
          </div>

          {/* Pro Insight info at the bottom of the history */}
          <div className="mt-5 pt-4 border-t border-slate-800">
            <div className="bg-blue-600/10 p-4 rounded-xl border border-blue-500/20">
              <p className="text-xs font-bold text-blue-400 flex items-center gap-1.5 uppercase tracking-wider">
                <Sparkles className="w-3.5 h-3.5" />
                Gemini Intelligence
              </p>
              <p className="text-[11px] text-slate-400 mt-1 leading-relaxed">
                Gemini translates contextually, preserving layout, code-blocks, emojis, list formatting, and technical markdown styles perfectly.
              </p>
            </div>
          </div>
        </section>

        {/* BENTO BLOCK 3: Developer Integration Guide (col-span-12 md:col-span-6 lg:col-span-6) */}
        <section className="col-span-12 md:col-span-6 lg:col-span-6 bg-slate-900 border border-slate-800 rounded-2xl p-6 flex flex-col justify-between shadow-xl min-h-[220px]" id="dev_bento">
          <div>
            <div className="flex items-center gap-3 mb-3">
              <div className="w-9 h-9 bg-purple-500/10 border border-purple-500/30 rounded-xl flex items-center justify-center text-purple-400">
                <BookOpen className="w-5 h-5" />
              </div>
              <div>
                <h4 className="text-white font-bold text-sm tracking-tight">Developer API Integration</h4>
                <p className="text-[11px] text-slate-500">Hook up official paid translation keys</p>
              </div>
            </div>
            
            <p className="text-xs text-slate-400 leading-relaxed mb-4">
              To wire official SDKs or third-party gateways (GCP / Azure), modify your server logic at <code>server.ts</code>. Keys remain securely hidden in the environment:
            </p>

            <div className="grid grid-cols-1 sm:grid-cols-2 gap-3 mb-1">
              <div className="p-3 bg-slate-950 rounded-xl border border-slate-800/60 text-[10px]">
                <span className="font-bold text-blue-400 uppercase tracking-wider text-[9px] block mb-1">Google Translate SDK</span>
                <code className="text-slate-400 font-mono">new TranslationServiceClient()</code>
              </div>
              <div className="p-3 bg-slate-950 rounded-xl border border-slate-800/60 text-[10px]">
                <span className="font-bold text-purple-400 uppercase tracking-wider text-[9px] block mb-1">Microsoft Azure API</span>
                <code className="text-slate-400 font-mono">api-version=3.0 & key</code>
              </div>
            </div>
          </div>

          <div className="pt-3 border-t border-slate-800/50 flex justify-between items-center text-xs text-slate-500">
            <span>Server: Express / CJS Bundle</span>
            <a href="https://cloud.google.com/translate/docs" target="_blank" rel="noreferrer" className="text-blue-400 hover:underline flex items-center gap-1 font-semibold text-[11px]">
              Docs <ExternalLink className="w-3 h-3" />
            </a>
          </div>
        </section>

        {/* BENTO BLOCK 4: Mobile Companion Lens Card (col-span-12 md:col-span-6 lg:col-span-6) */}
        <section className="col-span-12 md:col-span-6 lg:col-span-6 bg-gradient-to-br from-blue-700 to-indigo-850 rounded-2xl p-6 relative overflow-hidden group shadow-xl flex flex-col justify-between min-h-[220px]" id="companion_bento">
          {/* Subtle Background SVG Glow decoration */}
          <div className="absolute right-0 bottom-0 translate-x-1/4 translate-y-1/4 w-44 h-44 rounded-full bg-white/5 group-hover:scale-110 transition-transform duration-500" />
          <Smartphone className="absolute -right-4 -bottom-4 w-32 h-32 text-blue-500/20 group-hover:rotate-6 group-hover:scale-105 transition-all duration-300" />

          <div className="relative z-10">
            <div className="inline-block px-2 py-0.5 rounded bg-white/20 text-white font-mono text-[9px] uppercase tracking-wider font-extrabold mb-3">
              Aura App Ecosystem
            </div>
            <h4 className="text-white font-extrabold text-lg tracking-tight font-display">Mobile Translation Lens</h4>
            <p className="text-blue-100/90 text-xs mt-1.5 leading-relaxed max-w-[280px]">
              Point your camera or use voice scan to translate signboards, menus, and conversational speech on the go offline.
            </p>
          </div>

          <div className="relative z-10 flex gap-2 pt-4">
            <div className="px-3 py-1 bg-white/10 hover:bg-white/15 rounded-full text-[10px] text-white font-semibold backdrop-blur-sm transition-colors border border-white/10">
              iOS Companion
            </div>
            <div className="px-3 py-1 bg-white/10 hover:bg-white/15 rounded-full text-[10px] text-white font-semibold backdrop-blur-sm transition-colors border border-white/10">
              Android Native
            </div>
          </div>
        </section>

      </main>

      {/* Footer Stats Bar */}
      <footer className="h-12 border-t border-slate-800/80 px-6 sm:px-8 flex items-center justify-between bg-slate-950 shrink-0 z-10" id="app_footer">
        <div className="flex items-center gap-4 sm:gap-6">
          <div className="flex items-center gap-2">
            <div className="w-2 h-2 rounded-full bg-green-500 animate-pulse" />
            <span className="text-[11px] text-slate-400 uppercase font-mono tracking-wider">Cloud Sync Active</span>
          </div>
          <span className="text-[11px] text-slate-700">|</span>
          <span className="text-[11px] text-slate-400 font-mono">Build ID: 2026-v4.2</span>
        </div>
        <div className="flex items-center gap-4 text-[11px] text-slate-500">
          <span>AuraTranslate v4.2.0</span>
          <span className="hidden sm:inline text-slate-700">|</span>
          <a href="#" className="hover:text-slate-300 transition-colors hidden sm:inline">Privacy Policy</a>
        </div>
      </footer>
    </div>
  );
}
