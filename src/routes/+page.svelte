<script lang="ts">
    import { onMount } from 'svelte';
    
    let timeLeft = $state("00:57");
    let currentProblem = $state("3 + 12 = 1_");
    let upcomingProblems = $state([
        "7 * 2 =",
        "4 / 2 =",
        "13 + 24 =",
        "45 - 9 ="
    ]);

    // canvas status
    let canvas: HTMLCanvasElement;
    let ctx: CanvasRenderingContext2D | null = null;
    let isDrawing = false;
    let submitTimeout: ReturnType<typeof setTimeout>;
    
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

    onMount(() => {
        initCanvas();
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

    function processImage() {
        console.log("pen input complete. preprocess starting...");
        // TODO: BoundingBox Extraction -> resize 28*28 -> ONNX
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
        
            <!-- Current Problem -->
            <div class="bg-gray-300 text-black px-4 py-2 w-full max-w-sm text-center">
                {currentProblem}
            </div>

            {#each upcomingProblems as problem}
                <div class="w-full max-w-sm text-center opacity-50">
                    {problem}
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
