# UNMIRACLE CALCULATION

**UNMIRACLE CALCULATION** is a fast-paced, browser-based arithmetic game powered by on-device AI handwriting recognition. Players must solve an endless stream of dynamically generated math problems by drawing the answers directly onto a digital canvas within a 60-second time limit.

## ⚙️ Core Functional Features

*   **In-Browser AI Inference (Zero Latency):** 
    Utilizes **ONNX Runtime Web (WASM)** to execute a lightweight Convolutional Neural Network (MNIST CNN) directly on the client side. This ensures real-time handwriting recognition without any server-side API calls or network delays.
*   **Infinite & Safe Problem Generation:** 
    A robust algorithmic generator creates endless arithmetic problems (`+`, `-`, `*`, `/`) under strict mathematical constraints:
    *   Guarantees non-negative results for subtraction.
    *   Employs reverse-calculation logic for division to ensure only clean, integer-based equations (zero-division prevention).
*   **Sequential Digit Validation:** 
    Handles multi-digit answers by validating user inputs sequentially. For an answer like `15`, the logic isolates and checks the first digit (`1`), updates the reactive state, and awaits the subsequent digit (`5`) to clear the problem.
*   **Reactive Game Loop & Telemetry:** 
    Leverages Svelte 5's Rune system (`$state`, `$derived`) to manage the game flow (Splash -> Play -> Game Over). It calculates real-time metrics including a 60-second countdown, solved count, and dynamic accuracy percentage based on attempted and skipped problems.
*   **Progressive Web App (PWA) Ready:** 
    Configured with a `manifest.json` and viewport controls to act as a standalone application. It supports device orientation changes and operates natively when added to the home screen on iOS and Android.

## 🛠 Tech Stack

*   **Framework:** Svelte 5 (SvelteKit)
*   **Language:** TypeScript
*   **AI / ML:** ONNX Runtime Web (`onnxruntime-web`)
*   **Styling:** TailwindCSS

## 🚀 Getting Started

### Prerequisites
Make sure you have Node.js installed on your machine.

### Installation
1. Clone the repository.
2. Install the dependencies:
   ```bash
   npm install

 * Ensure your pre-trained AI model (mnist_cnn_single.onnx) is placed in the static/ directory of the project.
 * Run the development server:
   npm run dev

 * Open your browser and navigate to http://localhost:5173.

# 📜 License
This project is licensed under the MIT License - see the LICENSE file for details.

# 🤖 AI Assistance Disclosure
The source code, logic structures, and architectural design within this project were developed and authored with the assistance of Gemini AI (Google).

---
