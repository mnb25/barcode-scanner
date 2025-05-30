<script>
    let barcodeData = $state(new Map());
    let detectedBarcode = $state(null); // Detected barcode from the camera
    let detectedData = $state(null); // Data linked to the detected barcode
    let scanning = $state(false); // Whether the camera is scanning
    let videoStream = $state(null); // Video stream for the camera

    // Handle CSV file upload
    function handleFileUpload(event) {
        const file = event.target.files[0];
        const reader = new FileReader();

        reader.onload = (e) => {
            barcodeData.clear();
            const text = e.target.result;
            const rows = text.split("\n").filter((row) => row.trim() !== "");
            const headers = rows[0].split(",");

            rows.slice(1).forEach((row) => {
                const columns = row.split(",");
                const barcode = columns[0];
                const data = {};

                headers.slice(1).forEach((header, index) => {
                    data[header] = columns[index + 1];
                });

                barcodeData.set(barcode, data);
            });
            barcodeData = new Map(barcodeData); // Ensure reactivity
        };

        reader.readAsText(file);
    }

    // Start scanning for barcodes using the Barcode Detection API
    async function startScanning() {
        detectedBarcode = null;
        detectedData = null;

        if (!("BarcodeDetector" in window)) {
            console.warn(
                "Barcode Detection API is not supported in this browser. Using polyfill.",
            );
            window["BarcodeDetector"] =
                barcodeDetectorPolyfill.BarcodeDetectorPolyfill;
        }

        try {
            const video = document.querySelector("video");
            videoStream = await navigator.mediaDevices.getUserMedia({
                video: { facingMode: "environment" },
            });
            video.srcObject = videoStream;
            scanning = true;

            const barcodeDetector = new BarcodeDetector({
                formats: ["upc_a", "upc_e", "code_128", "ean_13", "ean_8"],
            });

            const scan = async () => {
                if (!scanning) return;

                try {
                    const barcodes = await barcodeDetector.detect(video);
                    if (barcodes.length > 0) {
                        detectedBarcode = barcodes[0].rawValue;
                        detectedData = barcodeData.get(detectedBarcode) || null;
                        stopScanning();
                        scanning = false;
                    }
                } catch (err) {
                    console.error("Error detecting barcode:", err);
                }

                if (scanning) {
                    requestAnimationFrame(scan);
                }
            };

            video.play();
            scan();
        } catch (err) {
            console.error("Error accessing camera:", err);
        }
    }

    // Stop scanning and release the camera
    function stopScanning() {
        if (videoStream) {
            const tracks = videoStream.getTracks();
            tracks.forEach((track) => track.stop());
            videoStream = null;
        }
        scanning = false;
    }
</script>

<main>
    <h1>Barcode Lookup</h1>

    <label for="upload-btn">
        1: Upload a CSV file that contains product data (first column must
        contain barcodes):
    </label>
    <input
        id="upload-btn"
        type="file"
        accept=".csv"
        onchange={handleFileUpload}
    />

    <label for="scan-btn">2: Point your camera at a barcode.</label>
    <button
        id="scan-btn"
        onclick={startScanning}
        style="display:{scanning ? 'none' : 'block'}"
    >
        Begin Scanning
    </button>

    {#if detectedBarcode}
        <h2>Result: {detectedBarcode}</h2>
        {#if detectedData}
            <h3>Linked Data:</h3>
            <ul>
                {#each Object.entries(detectedData) as [key, value]}
                    <li><strong>{key}:</strong> {value}</li>
                {/each}
            </ul>
        {:else}
            <p>No data linked to this barcode.</p>
        {/if}
    {/if}

    <video
        autoplay
        muted
        playsinline
        style="display:{scanning ? 'block' : 'none'}"
    ></video>
</main>

<style>
    * {
        font-family: sans-serif;
    }
    video {
        max-width: 10rem;
        margin-top: 0.5rem;
    }
    input[type="file"],
    button {
        display: block;
        margin: 0.5rem 0 1rem 0;
    }
</style>
