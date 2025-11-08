
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Engineer Text Summary Generator</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Configure Tailwind for Inter font -->
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                    colors: {
                        'primary-blue': '#1D4ED8',
                        'secondary-gray': '#E5E7EB',
                    }
                }
            }
        }
    </script>
    <style>
        /* Custom styles for better mobile readability and interface */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #F3F4F6;
        }
        /* Custom loading spinner */
        .spinner {
            border-top-color: #3B82F6;
            -webkit-animation: spin 1s linear infinite;
            animation: spin 1s linear infinite;
        }

        @-webkit-keyframes spin {
            0% { -webkit-transform: rotate(0deg); }
            100% { -webkit-transform: rotate(360deg); }
        }

        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body class="p-4 sm:p-8">

    <div class="max-w-4xl mx-auto bg-white shadow-2xl rounded-xl p-6 sm:p-10">
        
        <h1 class="text-3xl sm:text-4xl font-extrabold text-primary-blue mb-2 text-center">
            Engineer Text Summary Generator
        </h1>
        <p class="text-center text-gray-500 mb-8">
            Generates a professional, structured summary with Symptoms, Cause, and Solution headings.
        </p>

        <!-- Input Section -->
        <div class="mb-6">
            <label for="engineerInput" class="block text-lg font-medium text-gray-700 mb-2">1. Engineer's Detailed Notes:</label>
            <textarea id="engineerInput" rows="8" class="w-full p-4 border border-gray-300 rounded-lg focus:ring-primary-blue focus:border-primary-blue transition duration-150" placeholder="Paste the full, detailed technical report here. For example: 'Client reported excessive vibration from the main hydraulic pump (Symptom). Upon inspection, I found the coupling bolts had sheared due to previous over-torqueing (Cause). I replaced the damaged coupling with a new, correctly rated unit and performed laser alignment checks to ensure optimal operation (Solution).'">
Client reported excessive vibration and high-pitched noise coming from the primary coolant loop motor, which was causing intermittent thermal shutdowns. Disassembly revealed that the stator winding insulation had failed due to ingress of moisture, causing a phase-to-ground short circuit. The motor was removed, stripped down, and a full rewind was performed with VPI insulation. The motor was reinstalled and tested under full load for 2 hours with stable temperature and current readings.
            </textarea>
            <p class="text-sm text-gray-500 mt-1">This text will be analyzed and structured for the customer invoice.</p>
        </div>

        <!-- Button -->
        <div class="mb-10 flex justify-center">
            <button id="generateButton" onclick="generateSummary()" class="flex items-center justify-center bg-primary-blue text-white font-bold py-3 px-8 rounded-xl hover:bg-blue-700 transition duration-300 shadow-lg focus:outline-none focus:ring-4 focus:ring-blue-300 active:scale-95">
                <span id="buttonText">Generate Structured Summary</span>
                <div id="loadingSpinner" class="hidden spinner h-5 w-5 border-4 border-t-white rounded-full ml-3"></div>
            </button>
        </div>

        <!-- Output Section -->
        <div>
            <label for="summaryOutput" class="block text-lg font-medium text-gray-700 mb-2">2. Structured Invoice Summary:</label>
            <textarea id="summaryOutput" rows="8" readonly class="w-full p-4 border border-green-500 bg-green-50 rounded-lg font-mono text-gray-800 focus:outline-none resize-none"></textarea>
            <p id="charCount" class="text-sm text-gray-500 mt-1 text-right">Characters: 0 / 470 (Includes headers and spaces)</p>
            <button onclick="copyToClipboard()" class="mt-2 w-full bg-green-500 text-white font-semibold py-2 rounded-lg hover:bg-green-600 transition duration-300 focus:outline-none focus:ring-2 focus:ring-green-300 active:scale-95">
                Copy to Clipboard
            </button>
            <p id="copyMessage" class="text-sm text-center text-green-700 mt-2 opacity-0 transition duration-300">Copied!</p>
        </div>

        <!-- Error/Message Box -->
        <div id="messageBox" class="mt-6 p-4 bg-red-100 border border-red-400 text-red-700 rounded-lg hidden" role="alert">
            <p id="errorMessage" class="font-medium"></p>
        </div>

        <!-- Attribution -->
        <p class="text-center text-gray-400 text-sm mt-8">Created by Andrew Swan</p>
    </div>

    <script>
        // --- CRITICAL STEP FOR YOUR GITHUB PAGES DEPLOYMENT ---
        // 🚨 ACTION REQUIRED: Replace "AIzaSyCzbKIxl1kag2hqRb5bPGqxUp_GlhwqKmY" with your actual key.
        // This key will be visible in the source code.
        const USER_API_KEY = "AIzaSyCzbKIxl1kag2hqRb5bPGqxUp_GlhwqKmY"; 
        
        // --- GEMINI API CONFIGURATION (Direct Call) ---
        const MODEL_NAME = 'gemini-2.5-flash-preview-09-2025';
        const API_URL = `https://generativelanguage.googleapis.com/v1beta/models/${MODEL_NAME}:generateContent`;

        // Elements
        const engineerInput = document.getElementById('engineerInput');
        const summaryOutput = document.getElementById('summaryOutput');
        const generateButton = document.getElementById('generateButton');
        const buttonText = document.getElementById('buttonText');
        const loadingSpinner = document.getElementById('loadingSpinner');
        const charCount = document.getElementById('charCount');
        const messageBox = document.getElementById('messageBox');
        const errorMessage = document.getElementById('errorMessage');
        const copyMessage = document.getElementById('copyMessage');

        // Update character count on input
        summaryOutput.addEventListener('input', updateCharCount);
        summaryOutput.addEventListener('propertychange', updateCharCount); 
        
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
        
        window.onload = updateCharCount;


        /**
         * Hides the error message box.
         */
        function hideMessage() {
            messageBox.classList.add('hidden');
        }

        /**
         * Displays an error message in the message box.
         * @param {string} message - The error message to display.
         */
        function showMessage(message) {
            errorMessage.textContent = message;
            messageBox.classList.remove('hidden');
        }

        /**
         * System instruction defining the LLM's persona and output format.
         */
        const systemPrompt = `You are a professional technical summarization engine for client invoicing. Your task is to analyze detailed engineering reports (mechanical, electrical, or hydraulic) and extract the Symptoms, Cause, and Solution.

        You MUST format the output into exactly three separate paragraphs, each preceded by a bold heading and ending with a period. Use the following structure precisely, with two newline characters (\n\n) between each section:
        
        **Symptoms**
        [A professional, concise summary of the symptom(s).]

        **Cause**
        [A professional, concise summary of the cause identified.]

        **Solution**
        [A professional, concise summary of the remedy applied.]

        The total generated text, including the bold headers, paragraphs, and all newlines/spaces, MUST NOT exceed 470 characters in length. This is a strict, hard limit. Prioritize extreme brevity while maintaining professionalism.`;

        /**
         * Handles the API call with exponential backoff for retries.
         * @param {object} payload - The full request payload.
         * @param {number} maxRetries - Maximum number of retries.
         * @returns {Promise<object>} The API response data.
         */
        async function fetchWithBackoff(payload, maxRetries = 3) {
            
            for (let i = 0; i < maxRetries; i++) {
                try {
                    const url = `${API_URL}?key=${USER_API_KEY}`;
                    const response = await fetch(url, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });

                    if (response.status === 401) {
                         throw new Error("API Key Invalid. Please check the key in the source code.");
                    }
                    if (!response.ok) {
                        // For non-OK responses, retry.
                        if (i < maxRetries - 1) {
                             const delay = Math.pow(2, i) * 1000 + Math.random() * 1000;
                             await new Promise(resolve => setTimeout(resolve, delay));
                             continue;
                        }
                        const errorBody = await response.json().catch(() => ({error: {message: 'Non-JSON error response.'}}));
                        throw new Error(`API Request failed with status ${response.status}: ${errorBody.error?.message || 'Unknown API error'}`);
                    }

                    return await response.json();

                } catch (error) {
                    if (i === maxRetries - 1) {
                        throw error; // Re-throw the error on the last attempt
                    }
                    // Continue to next retry
                }
            }
        }

        /**
         * Main function to generate the summary by calling the Gemini API directly.
         */
        async function generateSummary() {
            hideMessage();
            const userQuery = engineerInput.value.trim();

            if (USER_API_KEY === "AIzaSyCzbKIxl1kag2hqRb5bPGqxUp_GlhwqKmY") {
                 showMessage("Error: API Key is missing. Please edit the index.html file and paste your key into the USER_API_KEY variable.");
                 return;
            }

            if (!userQuery) {
                showMessage("Please paste the engineer's technical notes into the input box before generating the summary.");
                return;
            }

            // UI state: Loading
            generateButton.disabled = true;
            buttonText.classList.add('hidden');
            loadingSpinner.classList.remove('hidden');
            summaryOutput.value = "Generating summary...";
            updateCharCount();

            // Reconstruct the final payload for the Gemini API
            const payload = {
                contents: [{ parts: [{ text: userQuery }] }],
                systemInstruction: {
                    parts: [{ text: systemPrompt }]
                },
            };

            try {
                const result = await fetchWithBackoff(payload);

                const candidate = result.candidates?.[0];

                if (candidate && candidate.content?.parts?.[0]?.text) {
                    const text = candidate.content.parts[0].text.trim();
                    summaryOutput.value = text;
                } else {
                    if (result.error && result.error.message) {
                         throw new Error(`API Error: ${result.error.message}`);
                    }
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
                // UI state: Finished
                generateButton.disabled = false;
                buttonText.classList.remove('hidden');
                loadingSpinner.classList.add('hidden');
                updateCharCount();
            }
        }

        /**
         * Copies the generated summary to the clipboard.
         */
        function copyToClipboard() {
            const textToCopy = summaryOutput.value;
            if (textToCopy && textToCopy !== "Generating summary..." && textToCopy !== "Error generating summary.") {
                // Use the document.execCommand('copy') fallback for compatibility
                try {
                    summaryOutput.select();
                    document.execCommand('copy');
                    
                    // Show confirmation message
                    copyMessage.classList.remove('opacity-0');
                    setTimeout(() => {
                        copyMessage.classList.add('opacity-0');
                    }, 2000);
                } catch (err) {
                    console.error('Could not copy text: ', err);
                }
            } else {
                showMessage("Nothing to copy or summary is still generating/in error state.");
            }
        }

    </script>
</body>
</html>

