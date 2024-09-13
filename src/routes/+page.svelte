<script>
    import {onMount, tick} from 'svelte';
    import JSZip from 'jszip';
    import Cropper from 'cropperjs';
    import 'cropperjs/dist/cropper.min.css';
    import Navbar from "../components/Navbar.svelte";

    let errorMessage = '';
    let previewUrl = '';
    let zipFileReady = false;
    let isLoading = false;
    let generatedIcons = [];
    let iconPrefix = 'icon';
    let selectedFormat = 'image/png';
    let manifestPreview = '';


    let name = 'Your App Name';
    let shortName = 'App';
    let file;
    let cropper;
    let imgElement;
    let zip;
    let fileInput;

    const formats = [
        {value: 'image/png', label: 'PNG'},
        {value: 'image/jpeg', label: 'JPEG'},
        {value: 'image/webp', label: 'WebP'}
    ];

    const sizes = [
        {size: 48, name: "48x48"},
        {size: 72, name: "72x72"},
        {size: 96, name: "96x96"},
        {size: 128, name: "128x128"},
        {size: 144, name: "144x144"},
        {size: 152, name: "152x152"},
        {size: 192, name: "192x192"},
        {size: 256, name: "256x256"},
        {size: 384, name: "384x384"},
        {size: 512, name: "512x512"}
    ];

    const handleFileUpload = async (e) => {
        isLoading = true;
        zip = new JSZip();
        file = e.target.files[0];
        if (!file || !file.type.startsWith('image/')) {
            errorMessage = "Please upload a valid image.";
            isLoading = false;
            return;
        }

        errorMessage = "";
        previewUrl = URL.createObjectURL(file);

        await tick();

        if (cropper) {
            cropper.destroy();
        }
        cropper = new Cropper(imgElement, {
            aspectRatio: 1,
            viewMode: 3,
            autoCropArea: 1,
        });

        isLoading = false;
    };

    const generateIcons = () => {
        isLoading = true;
        const canvasData = cropper.getCroppedCanvas();

        let icons = [];
        for (let {size, name} of sizes) {
            const resizedImage = resizeImage(canvasData, size, size, selectedFormat);
            const iconName = `${iconPrefix}-${name}.${getFormatExtension(selectedFormat)}`;
            const iconPath = `icons/${iconName}`;
            zip.folder("icons").file(iconName, resizedImage.split(",")[1], {base64: true});
            icons.push({dataUrl: resizedImage, name: iconName, path: iconPath});
        }
        generatedIcons = icons;
        generateManifest();
        zipFileReady = true;
        isLoading = false;
    };

    const resizeImage = (canvasData, width, height, format) => {
        const canvas = document.createElement("canvas");
        canvas.width = width;
        canvas.height = height;
        const ctx = canvas.getContext("2d");
        ctx.drawImage(canvasData, 0, 0, width, height);
        return canvas.toDataURL(format);
    };

    const getFormatExtension = (format) => {
        switch (format) {
            case 'image/png':
                return 'png';
            case 'image/jpeg':
                return 'jpg';
            case 'image/webp':
                return 'webp';
            default:
                return 'png';
        }
    };

    const generateManifest = () => {
        const manifest = {
            name: name,
            short_name: shortName,
            icons: generatedIcons.map(icon => ({
                src: icon.path,  // icons klasörüne göre güncellendi
                sizes: icon.name.match(/\d+x\d+/)[0],
                type: selectedFormat
            })),
            start_url: "/",
            display: "standalone",
            background_color: "#ffffff",
            theme_color: "#000000"
        };

        manifestPreview = JSON.stringify(manifest, null, 2);

        const manifestBlob = new Blob([manifestPreview], {type: 'application/json'});
        zip.file("manifest.json", manifestBlob);  // manifest dışarıda kalacak
    };

    const handleDownload = () => {
        if (zipFileReady) {
            zip.generateAsync({type: "blob"}).then((blob) => {
                const zipURL = URL.createObjectURL(blob);
                const a = document.createElement("a");
                a.href = zipURL;
                a.download = "icons_and_manifest.zip";
                a.click();
            });
        }
    };

    const handleDragOver = (e) => {
        e.preventDefault();
    };

    const handleDrop = (e) => {
        e.preventDefault();
        const dt = e.dataTransfer;
        const files = dt.files;
        handleFileUpload({target: {files}});
    };
</script>


<Navbar/>

<div class="min-h-screen grid grid-cols-1 lg:grid-cols-2 gap-6 p-4 lg:p-20">
    <div class="mockup-code overflow-y-auto max-h-[33rem]">
        {#if manifestPreview}
            <pre data-prefix="$"><code>{manifestPreview}</code></pre>
        {/if}
    </div>

    <div class="flex flex-col items-center gap-4">
        {#if errorMessage}
            <div class="alert alert-error shadow-lg w-full">
                <div>{errorMessage}</div>
            </div>
        {/if}

        <div
                class="upload-area w-full border-dashed border-2 border-base-400 bg-base-100 text-base-content hover:bg-base-200 transition-all p-4 rounded-lg"
                on:click={() => fileInput.click()}
                on:drop={handleDrop}
                on:dragover={handleDragOver}
        >
            <p class="text-lg font-semibold text-center">Drag and drop your image here</p>
        </div>


        <input type="file" class="hidden" accept="image/*" on:change={handleFileUpload} bind:this={fileInput}/>

        {#if previewUrl}
            <div class="flex flex-col lg:flex-row gap-6">
                <div class="w-48 max-w-xs rounded-lg shadow-lg overflow-hidden">
                    <img bind:this={imgElement} src={previewUrl} class="rounded-lg shadow-md" alt="Preview"/>
                </div>

                {#if generatedIcons.length > 0}
                    <div class="stack">
                        {#each [...generatedIcons].reverse() as icon}
                            <div class="icon-card flex flex-col items-center justify-center p-3 bg-white rounded-lg w-48 h-48 shadow">
                                <img src={icon.dataUrl} alt={icon.name} class="mask mask-squircle"/>
                                <p class="text-xs text-center mt-1">{icon.name}</p>
                            </div>
                        {/each}
                    </div>
                {/if}
            </div>
        {/if}

        <div class="grid grid-cols-1 lg:grid-cols-2 gap-4 w-full">
            <div class="form-control">
                <label class="label">
                    <span class="label-text">App Name</span>
                </label>
                <input type="text" class="input input-bordered w-full" bind:value={name}/>
            </div>
            <div class="form-control">
                <label class="label">
                    <span class="label-text">App Short Name</span>
                </label>
                <input type="text" class="input input-bordered w-full" bind:value={shortName}/>
            </div>
        </div>

        <div class="grid grid-cols-1 lg:grid-cols-2 gap-4 w-full">
            <div class="form-control">
                <label class="label">
                    <span class="label-text">Icon Prefix</span>
                </label>
                <input type="text" class="input input-bordered w-full" bind:value={iconPrefix}/>
            </div>
            <div class="form-control">
                <label class="label">
                    <span class="label-text">Select Format</span>
                </label>
                <select class="select select-bordered w-full" bind:value={selectedFormat}>
                    {#each formats as fmt}
                        <option value={fmt.value}>{fmt.label}</option>
                    {/each}
                </select>
            </div>
        </div>

        {#if generatedIcons.length > 0}
            <div class="grid grid-cols-1 lg:grid-cols-2 gap-4 w-full">
                <button class="btn btn-primary" on:click={generateIcons}>Generate Icons & Manifest</button>
                <button class="btn btn-secondary" on:click={handleDownload}>Download Icons & Manifest</button>
            </div>
        {:else}
            <button class="btn btn-primary w-full" on:click={generateIcons}>Generate Icons & Manifest</button>
        {/if}

        {#if isLoading}
            <progress class="progress w-full"></progress>
        {/if}

    </div>
</div>
