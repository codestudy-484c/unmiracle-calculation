<script lang="ts">
    import { onMount } from 'svelte';
    import * as ort from 'onnxruntime-web';
    
    ort.env.wasm.wasmPaths = '/';
    ort.env.wasm.numThreads = 1;
    
    type MathProblem = {
        equation: string;
        fullAnswer: string;
        currentIndex: number;
    };
    
    let timeLeft = $state("00:57");
    let currentProblem = $state<MathProblem>({ equation: "Now Loading...", fullAnswer: "0", currentIndex: 0 });
    let upcomingProblems = $state<MathProblem[]>([]);
    let previousProblem = $state<MathProblem | null>(null);

    let displayAnswer = $derived(
        currentProblem.fullAnswer.substring(0, currentProblem.currentIndex) + "_".repeat(currentProblem.fullAnswer.length - currentProblem.currentIndex)
    );
    
    // canvas status
    let canvas: HTMLCanvasElement;
    let ctx: CanvasRenderingContext2D | null = null;
    let isDrawing = false;
    let submitTimeout: ReturnType<typeof setTimeout>;

    let session: ort.InferenceSession | null = null;

    function generateRandomProblem(): MathProblem {
        const ops = ['+', '-', '*', '/'];
        const op = ops[Math.floor(Math.random() * ops.length)];

        let num1 = 0, num2 = 0, result = 0;
        let isValid = false;
        while (!isValid) {
            switch(op) {
                case '+':
                    num1 = Math.floor(Math.random() * 40) + 1;
                    num2 = Math.floor(Math.random() * 40) + 1;
                    result = num1 + num2;
                    isValid = true;
                    break;
                case '-':
                    num1 = Math.floor(Math.random() * 90) + 10;
                    num2 = Math.floor(Math.random() * 90) + 1;
                    result = num1 - num2;

                    if (result >= 0) {
                        isValid = true;
                    }
                    break;
                case '*':
                    num1 = Math.floor(Math.random() * 9) + 1;
                    num2 = Math.floor(Math.random() * 9) + 1;
                    result = num1 * num2;
                    isValid = true;
                    break;
                case '/':
                    num2 = Math.floor(Math.random() * 9) + 1;
                    result = Math.floor(Math.random() * 10);
    
                    num1 = num2 * result;
    
                    if (num2 > 0 && result >= 0 && Number.isInteger(result)) {
                        isValid = true;
                    }
                    break;
            }
        }
    
        return {
            equation: `${num1} ${op} ${num2} =`,
            fullAnswer: result.toString(),
            currentIndex: 0
        };
    }

    function setupInitialProblems() {
        currentProblem = generateRandomProblem();
        upcomingProblems = [
            generateRandomProblem(), generateRandomProblem(), generateRandomProblem()
        ];
    }

    async function loadModel() {
        try {
            console.log("Model and WASM runtime loading...");

            ort.env.wasm.wasmPaths = '/';
            ort.env.wasm.numThreads = 1

            session = await ort.InferenceSession.create('/mnist_cnn_single.onnx', {
                executionProviders: ['wasm']
            });
            console.log("ONNX model load successfully");

            setupInitialProblems();
        } catch (e) {
            console.error("ONNX model load failed:", e);
        }
    }
    
    function initCanvas() {
        if (!canvas) return;

        // for Apple Device Retina pixel scale
        const dpr = window.devicePixelRatio || 1;
        const rect = canvas.parentElement?.getBoundingClientRect() || { width: 300, height: 300 };

        canvas.width = rect.width * dpr;
        canvas.height = rect.height * dpr;

        ctx = canvas.getContext('2d', { willReadFrequently: true });
        if (ctx) {
            ctx.scale(dpr, dpr);
            ctx.lineCap = 'round';
            ctx.lineJoin = 'round';
            ctx.lineWidth = 15;

            clearCanvas();
        }
    }

    onMount(async () => {
        initCanvas();
        await loadModel();
    });

    // start drawing
    function startDrawing(e: PointerEvent) {
        if (!ctx) return;
        isDrawing = true;

        clearTimeout(submitTimeout);
        ctx.beginPath();
        ctx.moveTo(e.offsetX, e.offsetY);
        
        // scroll prevent
        if (e.cancelable) e.preventDefault();
    }
    
    // drawing (line)
    function draw(e: PointerEvent) {
        if (!isDrawing || !ctx) return;
        ctx.lineTo(e.offsetX, e.offsetY);
        ctx.strokeStyle = 'white';
        ctx.stroke();
        if (e.cancelable) e.preventDefault();
    }

    function stopDrawing() {
        if (!isDrawing) return;
        isDrawing = false;

        // wait 0.6s to check complete input
        submitTimeout = setTimeout(() => {
            processImage();
        }, 600);
    }

    function clearCanvas() {
        if (!ctx || !canvas) return;
        const rect = canvas.parentElement?.getBoundingClientRect();
        if (rect) {
            ctx.fillStyle = 'black';
            ctx.fillRect(0, 0, rect.width, rect.height);
        }
    }

    async function processImage() {
        const currentSession = session;

        // optional binding
        if (!ctx || !canvas || !currentSession) {
            console.warn("No canvas/onnx session");
            return;
        }

        if (!ctx || !canvas) return;
        
        const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        const data = imgData.data;

        // find bounding box of the handwritten text 
        let minX = canvas.width, minY = canvas.height, maxX = 0, maxY = 0;
        let hasPixel = false;

        for (let y = 0; y < canvas.height; y++) {
            for (let x = 0; x < canvas.width; x++) {
                const alpha = data[(y * canvas.width + x) * 4 + 3];
                const red = data[(y * canvas.width + x) * 4];
                if (alpha > 0 && red > 10) {
                    minX = Math.min(minX, x);
                    minY = Math.min(minY, y);
                    maxX = Math.max(maxX, x);
                    maxY = Math.max(maxY, y);
                    hasPixel = true;
                }
            }
        }
        
        if (!hasPixel) return;
        
        // Crop area
        const boxWidth = maxX - minX;
        const boxHeight = maxY - minY;

        // tempCanvas for conv to 28*28
        const tempCanvas = document.createElement('canvas');
        tempCanvas.width = 28;
        tempCanvas.height = 28;
        const tempCtx = tempCanvas.getContext('2d');
        if (!tempCtx) return;
        
        // recolor bg with black
        tempCtx.fillStyle = 'black';
        tempCtx.fillRect(0, 0, 28, 28);

        // Resize with aspect-ratio for 20x20 (margin 4px)
        const scale = 20 / Math.max(boxWidth, boxHeight);
        const scaledWidth = boxWidth * scale;
        const scaledHeight = boxHeight * scale;

        // align center
        const dx = (28 - scaledWidth) / 2;
        const dy = (28 - scaledHeight) / 2;
        
        // redraw
        tempCtx.drawImage(
            canvas,
            minX, minY, boxWidth, boxHeight,
            dx, dy, scaledWidth, scaledHeight
        );
    
        // Extract pixel data and normalization (0.1307, 0.3081)
        const resizedData= tempCtx.getImageData(0, 0, 28, 28).data;
        const tensorData = new Float32Array(28 * 28);

        for (let i = 0; i < (28 * 28); i++) {
            const pixelVal = resizedData[i * 4] / 255.0;
            tensorData[i] = (pixelVal - 0.1307) / 0.3081;
        }
        
        // ONNX model inference 
        try {
            const tensor = new ort.Tensor('float32', tensorData, [1, 1, 28, 28]);
            const feeds = { input: tensor };
            const results = await currentSession.run(feeds);
            const output = results.output.data;

            let maxIndex = 0;
            let maxValue = output[0];
            for (let i = 1; i < output.length; i++) {
                if (output[i] > maxValue) {
                    maxValue = output[i];
                    maxIndex = i;
                }
            }

            let targetDigit = parseInt(currentProblem.fullAnswer[currentProblem.currentIndex], 10);
            console.log(`Think: [ ${maxIndex} ] | Answer: ${targetDigit}`);
            
            if (maxIndex === targetDigit) {
                console.log("Correct");
                currentProblem.currentIndex++;

                if (currentProblem.currentIndex >= currentProblem.fullAnswer.length) {
                   console.log("All Correct");

                    previousProblem = currentProblem;

                   currentProblem = upcomingProblems.shift() || generateRandomProblem();
                   upcomingProblems.push(generateRandomProblem());
                }
                clearCanvas();
            } else {
                console.log("Incorrect. Retry.");
                clearCanvas();
            }

        } catch (error) {
            console.error("An error occured:", error);
        }
        
        
    }

</script>

<svelte:window onresize={initCanvas} />

<main class="select-none min-h-screen bg-black text-gray-300 font-mono flex flex-col landscape:flex-row selection:bg-gray-300 selection:text-black pt-[env(safe-area-inset-top)] pb-[env(safe-area-inset-bottom)] pl-[env(safe-area-inset-left)] pr-[env(safe-area-inset-right)]">

    <!--Left/Top area: timer, problems feed-->

    <section class="w-full landscape:w-1/2 flex flex-col border-b landscape:border-b-0 landscape:border-r border-gray-300 p-4 relative">
        <!--timer-->
        <div class="text-center text-3xl mb-8 mt-4 tracking-widest">
            {timeLeft}    
        </div>
        
        <!--List of Problems aligned in center-->
        <div class="flex-1 flex flex-col justify-center items-center text-2xl landscape:text-3xl space-y-4">
            <!-- Previous problem -->
            {#if previousProblem}
                <div class="text-gray-400 w-full max-w-sm text-center opacity-50">
                    {previousProblem.equation} <span class="text-green-400 opacity-80 font-bold">{previousProblem.fullAnswer}</span>
                </div>
            {/if}

            <!-- Current Problem -->
            <div class="bg-gray-300 text-black px-4 py-2 w-full max-w-sm text-center">
                {currentProblem.equation} <span class="font-bold">{displayAnswer}</span>
            </div>

            {#each upcomingProblems as problem}
                <div class="w-full max-w-sm text-center opacity-50">
                    {problem.equation}
                </div>
            {/each}

        </div>
    </section>

    <!--Right/Bottom area: canvas and control buttons-->
    <section class="w-full landscape:w-1/2 flex flex-col items-center justify-center p-8">
        <!--Input canvas container-->
        <div class="aspect-square w-full max-w-xs landscape:max-w-md border border-gray-300 bg-black/50 relative flex items-center justify-center touch-none">
            <canvas
                bind:this={canvas}
                onpointerdown={startDrawing}
                onpointermove={draw}
                onpointerup={stopDrawing}
                onpointerout={stopDrawing}
                onpointercancel={stopDrawing}
                class="w-full h-full absolute inset-0 cursor-crosshair"
            ></canvas>
        </div>

        <!--Action Button-->
        <div class="flex gap-8 mt-8 text-xl">
            <!-- BS -->
            <button onclick={clearCanvas} class="border border-gray-300 px-6 py-2 hover:bg-gray-300 hover:text-black transition-colors focus:outline-none">
                [ BS ]
            </button>

            <!-- skip -->
            <button class="border border-gray-300 px-6 py-2 hover:bg-gray-300 hover:text-black transition-colors focus:outline-none">
                [ SKIP ]
            </button>

        </div>
    </section>
</main>
