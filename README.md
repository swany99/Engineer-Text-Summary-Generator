<script>
// --- SECURITY ---
// Rotate your leaked key and replace below. Restrict by HTTP referrer (your GitHub Pages domain)
// and restrict to "Generative Language API". Consider moving calls to a tiny server proxy later.
const USER_API_KEY = "YOUR_RESTRICTED_BROWSER_KEY";

// Model: Gemini 2.5 Flash
const MODEL_NAME = "gemini-2.5-flash";

// Generative Language API v1 endpoint (browser-friendly)
const API_URL = `https://generativelanguage.googleapis.com/v1/models/${MODEL_NAME}:generateContent`;

// Elements
const engineerInput  = document.getElementById('engineerInput');
const summaryOutput  = document.getElementById('summaryOutput');
const generateButton = document.getElementById('generateButton');
const buttonText     = document.getElementById('buttonText');
const loadingSpinner = document.getElementById('loadingSpinner');
const charCount      = document.getElementById('charCount');
const messageBox     = document.getElementById('messageBox');
const errorMessage   = document.getElementById('errorMessage');
const copyMessage    = document.getElementById('copyMessage');

// Character counter
summaryOutput.addEventListener('input', updateCharCount);
summaryOutput.addEventListener('propertychange', updateCharCount);
window.onload = updateCharCount;

function updateCharCount() {
  const currentLength = summaryOutput.value.length;
  charCount.textContent = `Characters: ${currentLength} / 470 (Includes headers and spaces)`;
  if (currentLength > 470) {
    charCount.classList.remove('text-gray-500');
    charCount.classList.add('text-red-500');
  } else {
    charCount.classList.add('text-gray-500');
    charCount.classList.remove('text-red-500');
  }
}

function hideMessage() { messageBox.classList.add('hidden'); }
function showMessage(message) { errorMessage.textContent = message; messageBox.classList.remove('hidden'); }

// System prompt
const systemPrompt =
`You are a professional technical summarization engine for client invoicing. Your task is to analyze detailed engineering reports (mechanical, electrical, or hydraulic) and extract the Symptoms, Cause, and Solution.

You MUST format the output into exactly three separate paragraphs, each preceded by a bold heading and ending with a period. Use the following structure precisely, with two newline characters (\\n\\n) between each section:

**Symptoms**
[A professional, concise summary of the symptom(s).]

**Cause**
[A professional, concise summary of the cause identified.]

**Solution**
[A professional, concise summary of the remedy applied.]

The total generated text, including the bold headers, paragraphs, and all newlines/spaces, MUST NOT exceed 470 characters in length. This is a strict, hard limit. Prioritize extreme brevity while maintaining professionalism.`;

// Exponential backoff fetch
async function fetchWithBackoff(payload, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const url = `${API_URL}?key=${USER_API_KEY}`;
      const response = await fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });

      if (response.status === 401) {
        throw new Error("API Key Invalid. Please check the key in the source code.");
      }
      if (!response.ok) {
        if (i < maxRetries - 1) {
          const delay = Math.pow(2, i) * 1000 + Math.random() * 1000;
          await new Promise(r => setTimeout(r, delay));
          continue;
        }
        const errorBody = await response.json().catch(() => ({ error: { message: 'Non-JSON error response.' } }));
        const errMsg = errorBody.error?.message || 'Unknown API error';
        const err = new Error(`API Request failed with status ${response.status}: ${errMsg}`);
        err._rawServerMessage = errMsg;
        throw err;
      }

      return await response.json();
    } catch (err) {
      if (i === maxRetries - 1) throw err;
      // continue retry loop
    }
  }
}

// Payload using systemInstruction (preferred if supported by the model/endpoint)
function buildPayloadWithSystemInstruction(userText) {
  return {
    systemInstruction: { parts: [{ text: systemPrompt }] },
    contents: [
      { role: "user", parts: [{ text: userText }] }
    ],
    generationConfig: {
      temperature: 0.2,
      maxOutputTokens: 300,
      responseMimeType: "text/plain"
    }
  };
}

// Fallback payload: inline the system prompt into the user message
function buildPayloadInlineSystem(userText) {
  const combined = `${systemPrompt}\n\n[Engineer Notes]\n${userText}`;
  return {
    contents: [
      { role: "user", parts: [{ text: combined }] }
    ],
    generationConfig: {
      temperature: 0.2,
      maxOutputTokens: 300,
      responseMimeType: "text/plain"
    }
  };
}

async function generateSummary() {
  hideMessage();
  const userQuery = engineerInput.value.trim();

  if (!USER_API_KEY || USER_API_KEY === "YOUR_GEMINI_API_KEY_HERE" || USER_API_KEY === "YOUR_RESTRICTED_BROWSER_KEY") {
    showMessage("Error: API Key is missing. Please edit the script and paste your restricted browser key.");
    return;
  }
  if (!userQuery) {
    showMessage("Please paste the engineer's technical notes into the input box before generating the summary.");
    return;
  }

  // UI: loading
  generateButton.disabled = true;
  buttonText.classList.add('hidden');
  loadingSpinner.classList.remove('hidden');
  summaryOutput.value = "Generating summary...";
  updateCharCount();

  let payload = buildPayloadWithSystemInstruction(userQuery);

  try {
    let result;
    try {
      result = await fetchWithBackoff(payload);
    } catch (err) {
      const msg = (err._rawServerMessage || "").toLowerCase();
      // Smart fallback when endpoint/model doesn't accept systemInstruction
      if (msg.includes('unknown name "systeminstruction"') || msg.includes("cannot find field")) {
        payload = buildPayloadInlineSystem(userQuery);
        result = await fetchWithBackoff(payload);
      } else {
        throw err;
      }
    }

    const candidate = result.candidates?.[0];
    if (candidate?.content?.parts?.[0]?.text) {
      let text = candidate.content.parts[0].text.trim();

      // Safety guard if the model overshoots the 470-char constraint
      if (text.length > 470) {
        const cut = text.lastIndexOf('.', 470);
        text = text.slice(0, cut > 0 ? cut + 1 : 470);
      }

      summaryOutput.value = text;
    } else {
      if (result.error?.message) throw new Error(`API Error: ${result.error.message}`);
      throw new Error("Received an unexpected or empty response from the API.");
    }
  } catch (error) {
    console.error("Summary Generation Error:", error);
    const displayMessage = error.message.includes("API Key Invalid")
      ? error.message
      : `Failed to generate summary. Please check your network connection and API key. Details: ${error.message}`;
    showMessage(displayMessage);
    summaryOutput.value = "Error generating summary.";
  } finally {
    generateButton.disabled = false;
    buttonText.classList.remove('hidden');
    loadingSpinner.classList.add('hidden');
    updateCharCount();
  }
}

// Clipboard helper
function copyToClipboard() {
  const textToCopy = summaryOutput.value;
  if (textToCopy && textToCopy !== "Generating summary..." && textToCopy !== "Error generating summary.") {
    if (navigator.clipboard?.writeText) {
      navigator.clipboard.writeText(textToCopy).then(() => {
        copyMessage.classList.remove('opacity-0');
        setTimeout(() => copyMessage.classList.add('opacity-0'), 2000);
      }).catch(err => console.error('Clipboard write failed: ', err));
    } else {
      try {
        summaryOutput.select();
        document.execCommand('copy');
        copyMessage.classList.remove('opacity-0');
        setTimeout(() => copyMessage.classList.add('opacity-0'), 2000);
      } catch (err) {
        console.error('Could not copy text: ', err);
      }
    }
  } else {
    showMessage("Nothing to copy or summary is still generating/in error state.");
  }
}

window.generateSummary = generateSummary;
window.copyToClipboard = copyToClipboard;
</script>
