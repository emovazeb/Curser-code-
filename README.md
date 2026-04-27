# Curser-code-import { useState, useRef, useEffect } from "react";

const SYSTEM_PROMPT = `You are an expert mobile app developer. When the user describes their app idea, respond with:
1. A brief description of the app concept (2-3 sentences)
2. A list of 5-7 core features
3. A suggested tech stack (React Native or Flutter recommended)
4. GPS + transit behavior that includes:
   - Nearby train and subway stops/stations
   - Transit lines and station names to use
   - A step-by-step route line from origin to destination
5. A basic file/folder structure
6. The full code for the main screen of the app in React Native

Format your response in clearly labeled sections using these exact headers:
## CONCEPT
## FEATURES
## TECH STACK
## GPS & TRANSIT
## FILE STRUCTURE
## MAIN SCREEN CODE

Keep code practical and ready to use. Use React Native with functional components and hooks.
When creating map code, prefer react-native-maps polyline usage for visual route lines.

Transit/GPS accuracy rules (strict):
- Do not invent station names, coordinates, routes, or line numbers.
- Use "NEEDS_VERIFICATION" placeholders whenever exact stop/route data cannot be guaranteed.
- In generated React Native code, fetch route/station data from a real provider (Google Maps Directions/Places API, Mapbox Directions + Geocoding, or local GTFS-RT backend), not hardcoded coordinates.
- Include validation notes that exact transit legs depend on live provider response/time.
- If user input is ambiguous, ask for clarification in the GPS & TRANSIT section before finalizing route guidance.

Wake-up trigger requirements:
- User sets a pre-arrival bubble/time-limit (minutes before arrival).
- App calculates ETA continuously and triggers wake alert exactly when ETA <= bubble.
- App tracks live distance-to-destination and refreshes UI continuously as location changes.
- Include debounce/guard logic so wake alert fires once per trip, not repeatedly.
- Include foreground + background strategy (location updates, local notifications, battery-safe polling).`;

const API_BASE_URL = "http://localhost:8787";
const SECTION_HEADERS = ["CONCEPT", "FEATURES", "TECH STACK", "GPS & TRANSIT", "FILE STRUCTURE", "MAIN SCREEN CODE"];

const TypewriterText = ({ text }) => {
  const [displayed, setDisplayed] = useState("");

  useEffect(() => {
    setDisplayed("");
    let i = 0;
    const interval = setInterval(() => {
      if (i < text.length) {
        setDisplayed(text.slice(0, i + 1));
        i += 1;
      } else {
        clearInterval(interval);
      }
    }, 8);

    return () => clearInterval(interval);
  }, [text]);

  return <span>{displayed}</span>;
};

const parseResponse = (text) => {
  const sections = {};
  const escapedHeaders = SECTION_HEADERS.map((h) => h.replace(/[.*+?^${}()|[\]\\]/g, "\\$&"));
  const matcher = new RegExp(
    `^##\\s*(${escapedHeaders.join("|")})\\s*$([\\s\\S]*?)(?=^##\\s*(?:${escapedHeaders.join("|")})\\s*$|$)`,
    "gmi"
  );

  for (const match of text.matchAll(matcher)) {
    const header = (match[1] || "").toUpperCase().trim();
    const content = (match[2] || "").trim();
    sections[header] = content;
  }

  if (Object.keys(sections).length === 0 && text.trim()) {
    sections.CONCEPT = text.trim();
  }

  return sections;
};

const validateSections = (sections) => {
  const missing = SECTION_HEADERS.filter((header) => !sections[header] || !sections[header].trim());
  return { isValid: missing.length === 0, missing };
};

const copyTextWithFallback = async (text) => {
  if (!text) return false;

  try {
    if (navigator.clipboard?.writeText) {
      await navigator.clipboard.writeText(text);
      return true;
    }
  } catch {
    // Fall through to legacy copy method.
  }

  try {
    const textarea = document.createElement("textarea");
    textarea.value = text;
    textarea.setAttribute("readonly", "");
    textarea.style.position = "fixed";
    textarea.style.top = "-9999px";
    textarea.style.left = "-9999px";
    document.body.appendChild(textarea);
    textarea.select();
    textarea.setSelectionRange(0, textarea.value.length);
    const copied = document.execCommand("copy");
    document.body.removeChild(textarea);
    return copied;
  } catch {
    return false;
  }
};

export default function AppBuilder() {
  const [idea, setIdea] = useState("");
  const [startPoint, setStartPoint] = useState("");
  const [destination, setDestination] = useState("");
  const [wakeBeforeMinutes, setWakeBeforeMinutes] = useState("10");
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);
  const [activeTab, setActiveTab] = useState("CONCEPT");
  const [error, setError] = useState(null);
  const [copied, setCopied] = useState(false);
  const [serverStatus, setServerStatus] = useState("checking");
  const abortRef = useRef(null);

  useEffect(() => {
    return () => {
      abortRef.current?.abort();
    };
  }, []);

  useEffect(() => {
    let mounted = true;

    const checkServer = async () => {
      try {
        const response = await fetch(`${API_BASE_URL}/health`);
        if (!mounted) return;
        setServerStatus(response.ok ? "online" : "offline");
      } catch {
        if (!mounted) return;
        setServerStatus("offline");
      }
    };

    checkServer();
    const interval = setInterval(checkServer, 10000);

    return () => {
      mounted = false;
      clearInterval(interval);
    };
  }, []);

  const buildApp = async () => {
    if (!idea.trim()) return;

    if (!destination.trim()) {
      setError("Enter a destination for accurate transit routing.");
      return;
    }

    const wakeLeadTime = Number(wakeBeforeMinutes);
    if (!Number.isFinite(wakeLeadTime) || wakeLeadTime <= 0) {
      setError("Enter a valid wake-up bubble in minutes (greater than 0).");
      return;
    }

    abortRef.current?.abort();
    const controller = new AbortController();
    abortRef.current = controller;

    setLoading(true);
    setResult(null);
    setError(null);

    try {
      const userPrompt = [
        idea.trim(),
        startPoint.trim() ? `Start location: ${startPoint.trim()}` : "Start location: NEEDS_VERIFICATION (user did not provide).",
        `Destination: ${destination.trim()}`,
        `Wake-up bubble/time-limit before arrival: ${wakeLeadTime} minutes.`,
        "Ensure transit output includes subway/train stations and the line of getting there.",
        "Accuracy mode: do not guess. Use NEEDS_VERIFICATION if any stop/line cannot be confirmed.",
        "Must include live distance tracking and active ETA updates.",
        "Must include code logic: trigger wake alert exactly once when ETA is less than or equal to the wake-up bubble."
      ].join("\n");

      const response = await fetch(`${API_BASE_URL}/api/build-app`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        signal: controller.signal,
        body: JSON.stringify({
          idea: userPrompt,
          systemPrompt: SYSTEM_PROMPT,
        }),
      });

      const data = await response.json().catch(() => ({}));
      if (!response.ok) {
        throw new Error(data.error || `Request failed (${response.status})`);
      }

      const text = data.text || "";
      if (!text.trim()) {
        throw new Error("No content returned from the model.");
      }

      const parsed = parseResponse(text);
      const validation = validateSections(parsed);

      if (!validation.isValid) {
        throw new Error(
          `Model response is missing required section(s): ${validation.missing.join(", ")}. Please retry with clearer origin/destination details.`
        );
      }

      setResult(parsed);
      setActiveTab("CONCEPT");
    } catch (e) {
      if (e.name === "AbortError") {
        setError("Request canceled.");
      } else if (e instanceof TypeError) {
        setError("Cannot reach local server on http://localhost:8787. Start the proxy and try again.");
      } else {
        setError(e.message || "Something went wrong. Please try again.");
      }
    } finally {
      if (abortRef.current === controller) {
        abortRef.current = null;
      }
      setLoading(false);
    }
  };

  const cancelBuild = () => {
    if (!loading) return;
    abortRef.current?.abort();
  };

  const copyCode = async () => {
    const code = result?.["MAIN SCREEN CODE"];
    if (!code) return;

    const success = await copyTextWithFallback(code);
    if (success) {
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    } else {
      setError("Could not copy automatically. Please copy the code manually.");
    }
  };

  const tabs = SECTION_HEADERS;
  const statusColor = serverStatus === "online" ? "#22c55e" : serverStatus === "offline" ? "#ef4444" : "#eab308";
  const statusLabel = serverStatus === "online" ? "ONLINE" : serverStatus === "offline" ? "OFFLINE" : "CHECKING";

  return (
    <div
      style={{
        minHeight: "100vh",
        background: "#0a0a0f",
        fontFamily: "'Courier New', monospace",
        color: "#e0e0e0",
        padding: "0",
        display: "flex",
        flexDirection: "column",
      }}
    >
      <div
        style={{
          borderBottom: "1px solid #1e1e2e",
          padding: "24px 32px",
          display: "flex",
          alignItems: "center",
          gap: "16px",
          background: "#0d0d1a",
          flexWrap: "wrap",
        }}
      >
        <div
          style={{
            width: 40,
            height: 40,
            background: "linear-gradient(135deg, #7c3aed, #2563eb)",
            borderRadius: 10,
            display: "flex",
            alignItems: "center",
            justifyContent: "center",
            fontSize: 20,
          }}
        >
          📱
        </div>

        <div>
          <div style={{ fontSize: 20, fontWeight: "bold", color: "#fff", letterSpacing: 2 }}>APP.BUILD</div>
          <div style={{ fontSize: 11, color: "#555", letterSpacing: 3, textTransform: "uppercase" }}>
            AI Mobile App Generator
          </div>
        </div>

        <div
          style={{
            marginLeft: 16,
            display: "flex",
            alignItems: "center",
            gap: 8,
            fontSize: 10,
            letterSpacing: 2,
            color: "#888",
          }}
        >
          <span style={{ width: 8, height: 8, borderRadius: "50%", background: statusColor }} />
          API {statusLabel}
        </div>

        <div style={{ marginLeft: "auto", display: "flex", gap: 6 }}>
          {["#ff5f57", "#febc2e", "#28c840"].map((c) => (
            <div key={c} style={{ width: 12, height: 12, borderRadius: "50%", background: c }} />
          ))}
        </div>
      </div>

      <div style={{ flex: 1, padding: "40px 32px", maxWidth: 900, margin: "0 auto", width: "100%", boxSizing: "border-box" }}>
        <div style={{ marginBottom: 32 }}>
          <div style={{ fontSize: 12, color: "#7c3aed", letterSpacing: 3, marginBottom: 12, textTransform: "uppercase" }}>
            &gt; Describe your app idea
          </div>

          <div style={{ position: "relative" }}>
            <textarea
              value={idea}
              onChange={(e) => setIdea(e.target.value)}
              onKeyDown={(e) => {
                if (e.key === "Enter" && (e.metaKey || e.ctrlKey)) {
                  e.preventDefault();
                  buildApp();
                }
              }}
              placeholder="e.g. A fitness tracker app that uses AI to suggest personalized workout plans based on user goals and history..."
              rows={4}
              style={{
                width: "100%",
                background: "#111120",
                border: "1px solid #2a2a3e",
                borderRadius: 8,
                color: "#e0e0e0",
                fontFamily: "inherit",
                fontSize: 14,
                padding: "16px",
                resize: "vertical",
                outline: "none",
                boxSizing: "border-box",
                lineHeight: 1.6,
                transition: "border-color 0.2s",
              }}
              onFocus={(e) => {
                e.target.style.borderColor = "#7c3aed";
              }}
              onBlur={(e) => {
                e.target.style.borderColor = "#2a2a3e";
              }}
            />
          </div>

          <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(180px, 1fr))", gap: 10, marginTop: 10 }}>
            <input
              value={startPoint}
              onChange={(e) => setStartPoint(e.target.value)}
              placeholder="Start station / location (optional)"
              style={{
                width: "100%",
                background: "#111120",
                border: "1px solid #2a2a3e",
                borderRadius: 8,
                color: "#e0e0e0",
                fontFamily: "inherit",
                fontSize: 13,
                padding: "10px 12px",
                outline: "none",
                boxSizing: "border-box",
              }}
            />

            <input
              value={destination}
              onChange={(e) => setDestination(e.target.value)}
              placeholder="Destination station / location"
              style={{
                width: "100%",
                background: "#111120",
                border: "1px solid #2a2a3e",
                borderRadius: 8,
                color: "#e0e0e0",
                fontFamily: "inherit",
                fontSize: 13,
                padding: "10px 12px",
                outline: "none",
                boxSizing: "border-box",
              }}
            />

            <input
              type="number"
              min="1"
              step="1"
              value={wakeBeforeMinutes}
              onChange={(e) => setWakeBeforeMinutes(e.target.value)}
              placeholder="Wake before (min)"
              style={{
                width: "100%",
                background: "#111120",
                border: "1px solid #2a2a3e",
                borderRadius: 8,
                color: "#e0e0e0",
                fontFamily: "inherit",
                fontSize: 13,
                padding: "10px 12px",
                outline: "none",
                boxSizing: "border-box",
              }}
            />
          </div>

          <div style={{ marginTop: 8, fontSize: 11, color: "#6b7280" }}>
            Use exact station names/full addresses and set your wake-up bubble (minutes before arrival).
          </div>

          <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginTop: 12, gap: 12, flexWrap: "wrap" }}>
            <div style={{ fontSize: 11, color: "#333" }}>Ctrl/Cmd + Enter to submit</div>
            <button
              onClick={loading ? cancelBuild : buildApp}
              disabled={!loading && !idea.trim()}
              style={{
                background: loading ? "#5a1a1a" : "linear-gradient(135deg, #7c3aed, #2563eb)",
                color: "#fff",
                border: "none",
                borderRadius: 6,
                padding: "10px 28px",
                fontFamily: "inherit",
                fontSize: 13,
                fontWeight: "bold",
                letterSpacing: 2,
                cursor: !loading && !idea.trim() ? "not-allowed" : "pointer",
                transition: "all 0.2s",
                textTransform: "uppercase",
                opacity: !loading && !idea.trim() ? 0.5 : 1,
              }}
              aria-disabled={!loading && !idea.trim()}
            >
              {loading ? "CANCEL BUILD" : "BUILD APP →"}
            </button>
          </div>
        </div>

        {loading && (
          <div style={{ textAlign: "center", padding: "60px 0" }}>
            <div style={{ fontSize: 13, color: "#7c3aed", letterSpacing: 3, marginBottom: 24 }}>GENERATING YOUR APP...</div>
            <div style={{ display: "flex", justifyContent: "center", gap: 8 }}>
              {[0, 1, 2].map((i) => (
                <div
                  key={i}
                  style={{
                    width: 8,
                    height: 8,
                    borderRadius: "50%",
                    background: "#7c3aed",
                    animation: `pulse 1.2s ease-in-out ${i * 0.2}s infinite`,
                  }}
                />
              ))}
            </div>
            <style>{`
              @keyframes pulse {
                0%, 80%, 100% { opacity: 0.2; transform: scale(0.8); }
                40% { opacity: 1; transform: scale(1.2); }
              }
            `}</style>
          </div>
        )}

        {error && (
          <div
            style={{
              background: "#1a0a0a",
              border: "1px solid #5a1a1a",
              borderRadius: 8,
              padding: 16,
              color: "#f87171",
              fontSize: 13,
            }}
          >
            {error}
          </div>
        )}

        {result && (
          <div>
            <div style={{ fontSize: 12, color: "#7c3aed", letterSpacing: 3, marginBottom: 16, textTransform: "uppercase" }}>
              &gt; App Blueprint Ready
            </div>

            <div style={{ display: "flex", gap: 4, marginBottom: 0, flexWrap: "wrap" }}>
              {tabs.map((tab) => (
                <button
                  key={tab}
                  onClick={() => setActiveTab(tab)}
                  style={{
                    background: activeTab === tab ? "#7c3aed" : "#111120",
                    color: activeTab === tab ? "#fff" : "#555",
                    border: `1px solid ${activeTab === tab ? "#7c3aed" : "#2a2a3e"}`,
                    borderBottom: activeTab === tab ? "1px solid #111120" : "1px solid #2a2a3e",
                    borderRadius: "6px 6px 0 0",
                    padding: "8px 14px",
                    fontFamily: "inherit",
                    fontSize: 10,
                    letterSpacing: 2,
                    cursor: "pointer",
                    textTransform: "uppercase",
                    transition: "all 0.15s",
                    marginBottom: -1,
                    zIndex: activeTab === tab ? 1 : 0,
                    position: "relative",
                  }}
                >
                  {tab === "MAIN SCREEN CODE" ? "CODE" : tab === "GPS & TRANSIT" ? "GPS" : tab}
                </button>
              ))}
            </div>

            <div
              style={{
                background: "#111120",
                border: "1px solid #2a2a3e",
                borderRadius: "0 6px 6px 6px",
                padding: 24,
                position: "relative",
              }}
            >
              {activeTab === "MAIN SCREEN CODE" && (
                <button
                  onClick={copyCode}
                  style={{
                    position: "absolute",
                    top: 16,
                    right: 16,
                    background: copied ? "#166534" : "#1e1e2e",
                    color: copied ? "#4ade80" : "#888",
                    border: `1px solid ${copied ? "#166534" : "#333"}`,
                    borderRadius: 4,
                    padding: "4px 12px",
                    fontFamily: "inherit",
                    fontSize: 10,
                    letterSpacing: 2,
                    cursor: "pointer",
                    textTransform: "uppercase",
                  }}
                >
                  {copied ? "COPIED ✓" : "COPY"}
                </button>
              )}

              {activeTab === "MAIN SCREEN CODE" ? (
                <pre
                  style={{
                    color: "#a5f3fc",
                    fontSize: 12,
                    lineHeight: 1.7,
                    overflow: "auto",
                    margin: 0,
                    whiteSpace: "pre-wrap",
                    wordBreak: "break-word",
                    paddingRight: 80,
                  }}
                >
                  {result[activeTab] || "No code generated."}
                </pre>
              ) : (
                <div
                  style={{
                    fontSize: 13,
                    lineHeight: 1.8,
                    color: "#c0c0d0",
                    whiteSpace: "pre-wrap",
                  }}
                >
                  <TypewriterText text={result[activeTab] || "No content."} />
                </div>
              )}
            </div>

            <div
              style={{
                marginTop: 24,
                padding: 16,
                background: "#0d0d1a",
                border: "1px solid #1e1e2e",
                borderRadius: 8,
                fontSize: 12,
                color: "#444",
              }}
            >
              <span style={{ color: "#7c3aed" }}>NEXT →</span> Copy the code, set up a React Native project with <span style={{ color: "#888" }}>npx create-expo-app MyApp</span>, and paste the main screen code to get started.
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
