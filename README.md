<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Image Smoothing Filter</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
        }
        canvas {
            max-width: 100%;
            height: auto;
            border-radius: 0.5rem;
            box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
        }
        .custom-file-input::-webkit-file-upload-button {
            visibility: hidden;
        }
        .custom-file-input::before {
            content: 'Select Image';
            display: inline-block;
            background: #4f46e5; /* indigo-600 */
            color: white;
            border: 1px solid #4f46e5;
            border-radius: 0.375rem;
            padding: 0.5rem 1rem;
            outline: none;
            white-space: nowrap;
            -webkit-user-select: none;
            cursor: pointer;
            font-weight: 500;
            transition: background-color 0.2s;
        }
        .custom-file-input:hover::before {
            background-color: #4338ca; /* indigo-700 */
        }
        .custom-file-input:active::before {
            background-color: #3730a3; /* indigo-800 */
        }
    </style>
</head>
<body class="bg-gray-100 text-gray-800 flex flex-col items-center justify-center min-h-screen p-4">

    <div class="w-full max-w-6xl bg-white rounded-xl shadow-lg p-6 md:p-8">
        <header class="text-center mb-8">
            <h1 class="text-3xl md:text-4xl font-bold text-gray-900">Image Smoothing Filter</h1>
            <p class="text-gray-600 mt-2">Upload an image to apply a smoothing filter and see the results.</p>
        </header>

        <!-- Controls Section -->
        <div class="bg-gray-50 p-6 rounded-lg mb-8 flex flex-col md:flex-row items-center justify-center gap-6">
            <!-- File Upload -->
            <div class="flex flex-col items-center">
                <label for="imageLoader" class="text-sm font-medium text-gray-700 mb-2">1. Upload Image</label>
                <input type="file" id="imageLoader" name="imageLoader" accept="image/*" class="custom-file-input text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0 file:text-sm file:font-semibold file:bg-indigo-50 file:text-indigo-700 hover:file:bg-indigo-100"/>
            </div>

            <!-- Filter Options -->
            <div class="flex flex-col items-center">
                <label for="filterSize" class="text-sm font-medium text-gray-700 mb-2">2. Filter Size</label>
                <select id="filterSize" class="bg-white border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-indigo-500 focus:border-indigo-500 block w-full p-2.5">
                    <option value="3">3×3 Neighborhood</option>
                    <option value="5">5×5 Neighborhood</option>
                    <option value="7">7×7 Neighborhood</option>
                    <option value="9">9×9 Neighborhood</option>
                </select>
            </div>
            
            <!-- Grayscale Checkbox -->
            <div class="flex items-center pt-6 md:pt-0">
                 <input id="grayscale" type="checkbox" class="w-4 h-4 text-indigo-600 bg-gray-100 border-gray-300 rounded focus:ring-indigo-500">
                 <label for="grayscale" class="ml-2 text-sm font-medium text-gray-900">Convert to Grayscale First</label>
            </div>

            <!-- Apply Button -->
            <div class="flex flex-col items-center">
                <label class="text-sm font-medium text-gray-700 mb-2 invisible">3. Apply</label>
                <button id="smoothButton" class="w-full md:w-auto bg-indigo-600 text-white font-bold py-2 px-6 rounded-lg hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 transition-colors disabled:bg-gray-400 disabled:cursor-not-allowed">
                    Smooth Image
                </button>
            </div>
        </div>

        <!-- Image Display Section -->
        <div class="grid grid-cols-1 md:grid-cols-2 gap-8">
            <!-- Original Image -->
            <div class="flex flex-col items-center">
                <h2 class="text-xl font-semibold mb-3">Original Image</h2>
                <div class="relative">
                    <canvas id="originalCanvas" class="bg-gray-200"></canvas>
                    <div id="pixelInfo" class="absolute bottom-0 left-0 bg-black bg-opacity-60 text-white text-xs p-2 rounded-tr-lg rounded-bl-lg pointer-events-none hidden">
                        RGBA: (0, 0, 0, 0)
                    </div>
                </div>
                 <p id="original-placeholder" class="mt-4 text-gray-500">Upload an image to see it here</p>
            </div>
            <!-- Smoothed Image -->
            <div class="flex flex-col items-center">
                <h2 class="text-xl font-semibold mb-3">Smoothed Image</h2>
                <canvas id="smoothedCanvas" class="bg-gray-200"></canvas>
                <p id="smoothed-placeholder" class="mt-4 text-gray-500">Your smoothed image will appear here</p>
            </div>
        </div>
    </div>

    <script>
        // --- DOM Element References ---
        const imageLoader = document.getElementById('imageLoader');
        const originalCanvas = document.getElementById('originalCanvas');
        const originalCtx = originalCanvas.getContext('2d', { willReadFrequently: true });
        const smoothedCanvas = document.getElementById('smoothedCanvas');
        const smoothedCtx = smoothedCanvas.getContext('2d');
        const smoothButton = document.getElementById('smoothButton');
        const filterSizeSelect = document.getElementById('filterSize');
        const grayscaleCheckbox = document.getElementById('grayscale');
        const pixelInfo = document.getElementById('pixelInfo');
        const originalPlaceholder = document.getElementById('original-placeholder');
        const smoothedPlaceholder = document.getElementById('smoothed-placeholder');

        let originalImageData = null;
        smoothButton.disabled = true;

        // --- Event Listeners ---

        /**
         * Handles the file input change event.
         * Reads the selected image file, creates an Image object, and draws it on the original canvas.
         */
        imageLoader.addEventListener('change', (e) => {
            const reader = new FileReader();
            reader.onload = (event) => {
                const img = new Image();
                img.onload = () => {
                    // Set canvas dimensions to match the image
                    originalCanvas.width = img.width;
                    originalCanvas.height = img.height;
                    smoothedCanvas.width = img.width;
                    smoothedCanvas.height = img.height;
                    
                    // Draw the image and store its pixel data
                    originalCtx.drawImage(img, 0, 0);
                    originalImageData = originalCtx.getImageData(0, 0, originalCanvas.width, originalCanvas.height);
                    
                    // Update UI
                    smoothButton.disabled = false;
                    originalPlaceholder.classList.add('hidden');
                    smoothedPlaceholder.classList.remove('hidden');
                    smoothedCtx.clearRect(0, 0, smoothedCanvas.width, smoothedCanvas.height); // Clear previous result
                };
                img.src = event.target.result;
            };
            if (e.target.files[0]) {
                reader.readAsDataURL(e.target.files[0]);
            }
        });

        /**
         * Handles the click event on the "Smooth Image" button.
         * Applies the selected filter to the original image data and displays the result.
         */
        smoothButton.addEventListener('click', () => {
            if (!originalImageData) {
                alert('Please upload an image first.');
                return;
            }

            let sourceData = originalImageData;
            
            // Apply grayscale if checked
            if (grayscaleCheckbox.checked) {
                sourceData = convertToGrayscale(originalImageData);
                // Also draw the grayscaled version on the original canvas for clarity
                originalCtx.putImageData(sourceData, 0, 0);
            } else {
                // If not grayscale, ensure the original image is drawn
                originalCtx.putImageData(originalImageData, 0, 0);
            }

            const filterSize = parseInt(filterSizeSelect.value);
            const smoothedData = applySmoothing(sourceData, filterSize);
            
            // Draw the smoothed image on the second canvas
            smoothedCtx.putImageData(smoothedData, 0, 0);
            smoothedPlaceholder.classList.add('hidden');
        });

        /**
         * Shows pixel RGBA values on mouse hover over the original canvas.
         */
        originalCanvas.addEventListener('mousemove', (e) => {
            if (!originalImageData) return;

            const rect = originalCanvas.getBoundingClientRect();
            const scaleX = originalCanvas.width / rect.width;
            const scaleY = originalCanvas.height / rect.height;
            
            const x = Math.floor((e.clientX - rect.left) * scaleX);
            const y = Math.floor((e.clientY - rect.top) * scaleY);

            const pixelIndex = (y * originalCanvas.width + x) * 4;
            const r = originalImageData.data[pixelIndex];
            const g = originalImageData.data[pixelIndex + 1];
            const b = originalImageData.data[pixelIndex + 2];
            const a = originalImageData.data[pixelIndex + 3];

            pixelInfo.textContent = `RGBA: (${r}, ${g}, ${b}, ${a})`;
            pixelInfo.classList.remove('hidden');
        });

        /**
         * Hides the pixel info box when the mouse leaves the canvas.
         */
        originalCanvas.addEventListener('mouseleave', () => {
            pixelInfo.classList.add('hidden');
        });


        // --- Image Processing Functions ---

        /**
         * Converts image data to grayscale.
         * @param {ImageData} imageData - The original image data.
         * @returns {ImageData} - The new grayscaled image data.
         */
        function convertToGrayscale(imageData) {
            const data = new Uint8ClampedArray(imageData.data);
            for (let i = 0; i < data.length; i += 4) {
                const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
                data[i] = avg;     // Red
                data[i + 1] = avg; // Green
                data[i + 2] = avg; // Blue
            }
            return new ImageData(data, imageData.width, imageData.height);
        }

        /**
         * Applies a smoothing (box blur) filter to image data.
         * @param {ImageData} sourceImageData - The image data to process.
         * @param {number} filterSize - The size of the neighborhood (e.g., 3 for 3x3).
         * @returns {ImageData} - The new smoothed image data.
         */
        function applySmoothing(sourceImageData, filterSize) {
            const src = sourceImageData.data;
            const width = sourceImageData.width;
            const height = sourceImageData.height;
            
            const destData = new Uint8ClampedArray(src.length);
            const destImageData = new ImageData(destData, width, height);
            const dest = destImageData.data;

            const radius = Math.floor(filterSize / 2);

            // Iterate over each pixel
            for (let y = 0; y < height; y++) {
                for (let x = 0; x < width; x++) {
                    let r = 0, g = 0, b = 0, a = 0;
                    let pixelCount = 0;

                    // Iterate over the neighborhood
                    for (let ny = -radius; ny <= radius; ny++) {
                        for (let nx = -radius; nx <= radius; nx++) {
                            const currentX = x + nx;
                            const currentY = y + ny;

                            // Check if the neighborhood pixel is within bounds
                            if (currentX >= 0 && currentX < width && currentY >= 0 && currentY < height) {
                                const i = (currentY * width + currentX) * 4;
                                r += src[i];
                                g += src[i + 1];
                                b += src[i + 2];
                                a += src[i + 3];
                                pixelCount++;
                            }
                        }
                    }

                    // Calculate the average and set the destination pixel value
                    const destIndex = (y * width + x) * 4;
                    dest[destIndex] = r / pixelCount;
                    dest[destIndex + 1] = g / pixelCount;
                    dest[destIndex + 2] = b / pixelCount;
                    dest[destIndex + 3] = a / pixelCount; // We average alpha too, though often it's just kept at 255
                }
            }
            return destImageData;
        }
    </script>
</body>
</html>
