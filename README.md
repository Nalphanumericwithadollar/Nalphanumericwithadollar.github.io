<html lang="en"> 
  <head><style>body {transition: opacity ease-in 0.2s; } 
body[unresolved] {opacity: 0; display: block; overflow: hidden; position: relative; } 
</style>
    <meta charset="utf-8">
    <title>Sporcle Picture Box Creator</title>
    <meta name="description" content="Easily create picture box images for sporcle">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/croppie/2.6.5/croppie.min.css">
<script>
    /**
 * hermite-resize - Canvas image resize/resample using Hermite filter with JavaScript.
 * @version v2.2.10
 * @link https://github.com/viliusle/miniPaint
 * @license MIT
 */
    /*
     * Hermite resize - fast image resize/resample using Hermite filter.
     * https://github.com/viliusle/Hermite-resize
     */
    function Hermite() {
        var cores;
        var workers_archive = [];
        var workerBlobURL;

        /**
         * contructor
         */
        this.init = function () {
            cores = navigator.hardwareConcurrency || 4;
        }();

        /**
         * Returns CPU cores count
         * 
         * @returns {int}
         */
        this.getCores = function () {
            return cores;
        };

        /**
         * Hermite resize. Detect cpu count and use best option for user.
         * 
         * @param {HtmlElement} canvas
         * @param {int} width
         * @param {int} height
         * @param {boolean} resize_canvas if true, canvas will be resized. Optional.
         * @param {boolean} on_finish finish handler. Optional.
         */
        this.resample_auto = function (canvas, width, height, resize_canvas, on_finish) {
            var cores = this.getCores();

            if (!!window.Worker && cores > 1) {
                //workers supported and we have at least 2 cpu cores - using multithreading
                this.resample(canvas, width, height, resize_canvas, on_finish);
            }
            else {
                //1 cpu version
                this.resample_single(canvas, width, height, true);
                if (on_finish != undefined) {
                    on_finish();
                }
            }
        };

        /**
         * Hermite resize. Resize actual image.
         * 
         * @param {string} image_id
         * @param {int} width
         * @param {int} height optional.
         * @param {int} percentages optional.
         * @param {string} multi_core optional.
         */
        this.resize_image = function (image_id, width, height, percentages, multi_core) {
            var img = document.getElementById(image_id);

            //create temp canvas
            var temp_canvas = document.createElement("canvas");
            temp_canvas.width = img.width;
            temp_canvas.height = img.height;
            var temp_ctx = temp_canvas.getContext("2d");

            //draw image
            temp_ctx.drawImage(img, 0, 0);

            //prepare size
            if (width == undefined && height == undefined && percentages != undefined) {
                width = img.width / 100 * percentages;
                height = img.height / 100 * percentages;
            }
            if (height == undefined) {
                var ratio = img.width / width;
                height = img.height / ratio;
            }
            width = Math.round(width);
            height = Math.round(height);

            var on_finish = function () {
                var dataURL = temp_canvas.toDataURL();
                img.width = width;
                img.height = height;
                img.src = dataURL;

                dataURL = null;
                temp_canvas = null;
            };

            //resize
            if (multi_core == undefined || multi_core == true) {
                this.resample(temp_canvas, width, height, true, on_finish);
            }
            else {
                this.resample_single(temp_canvas, width, height, true);
                on_finish();
            }
        };

        /**
         * Hermite resize, multicore version - fast image resize/resample using Hermite filter.
         * 
         * @param {HtmlElement} canvas
         * @param {int} width
         * @param {int} height
         * @param {boolean} resize_canvas if true, canvas will be resized. Optional.
         * @param {boolean} on_finish finish handler. Optional.
         */
        this.resample = function (canvas, width, height, resize_canvas, on_finish) {
            var width_source = canvas.width;
            var height_source = canvas.height;
            width = Math.round(width);
            height = Math.round(height);
            var ratio_h = height_source / height;

            //stop old workers
            if (workers_archive.length > 0) {
                for (var c = 0; c < cores; c++) {
                    if (workers_archive[c] != undefined) {
                        workers_archive[c].terminate();
                        delete workers_archive[c];
                    }
                }
            }
            workers_archive = new Array(cores);
            var ctx = canvas.getContext("2d");

            //prepare source and target data for workers
            var data_part = [];
            var block_height = Math.ceil(height_source / cores / 2) * 2;
            var end_y = -1;
            for (var c = 0; c < cores; c++) {
                //source
                var offset_y = end_y + 1;
                if (offset_y >= height_source) {
                    //size too small, nothing left for this core
                    continue;
                }

                end_y = offset_y + block_height - 1;
                end_y = Math.min(end_y, height_source - 1);

                var current_block_height = block_height;
                current_block_height = Math.min(block_height, height_source - offset_y);

                //console.log('source split: ', '#'+c, offset_y, end_y, 'height: '+current_block_height);

                data_part[c] = {};
                data_part[c].source = ctx.getImageData(0, offset_y, width_source, block_height);
                data_part[c].target = true;
                data_part[c].start_y = Math.ceil(offset_y / ratio_h);
                data_part[c].height = current_block_height;
            }

            //clear and resize canvas
            if (resize_canvas === true) {
                canvas.width = width;
                canvas.height = height;
            }
            else {
                ctx.clearRect(0, 0, width_source, height_source);
            }

            //start
            var workers_in_use = 0;
            for (var c = 0; c < cores; c++) {
                if (data_part[c] == undefined) {
                    //no job for this worker
                    continue;
                }

                workers_in_use++;
                var my_worker = new Worker(workerBlobURL);
                workers_archive[c] = my_worker;

                my_worker.onmessage = function (event) {
                    workers_in_use--;
                    var core = event.data.core;
                    workers_archive[core].terminate();
                    delete workers_archive[core];

                    //draw
                    var height_part = Math.ceil(data_part[core].height / ratio_h);
                    data_part[core].target = ctx.createImageData(width, height_part);
                    data_part[core].target.data.set(event.data.target);
                    ctx.putImageData(data_part[core].target, 0, data_part[core].start_y);

                    if (workers_in_use <= 0) {
                        //finish
                        if (on_finish != undefined) {
                            on_finish();
                        }
                    }
                };
                var objData = {
                    width_source: width_source,
                    height_source: data_part[c].height,
                    width: width,
                    height: Math.ceil(data_part[c].height / ratio_h),
                    core: c,
                    source: data_part[c].source.data.buffer,
                };
                my_worker.postMessage(objData, [objData.source]);
            }
        };

        // Build a worker from an anonymous function body - purpose is to avoid separate file
        workerBlobURL = window.URL.createObjectURL(new Blob(['(',
            function () {
                //begin worker
                onmessage = function (event) {
                    var core = event.data.core;
                    var width_source = event.data.width_source;
                    var height_source = event.data.height_source;
                    var width = event.data.width;
                    var height = event.data.height;

                    var ratio_w = width_source / width;
                    var ratio_h = height_source / height;
                    var ratio_w_half = Math.ceil(ratio_w / 2);
                    var ratio_h_half = Math.ceil(ratio_h / 2);

                    var source = new Uint8ClampedArray(event.data.source);
                    var source_h = source.length / width_source / 4;
                    var target_size = width * height * 4;
                    var target_memory = new ArrayBuffer(target_size);
                    var target = new Uint8ClampedArray(target_memory, 0, target_size);
                    //calculate
                    for (var j = 0; j < height; j++) {
                        for (var i = 0; i < width; i++) {
                            var x2 = (i + j * width) * 4;
                            var weight = 0;
                            var weights = 0;
                            var weights_alpha = 0;
                            var gx_r = 0;
                            var gx_g = 0;
                            var gx_b = 0;
                            var gx_a = 0;
                            var center_y = j * ratio_h;

                            var xx_start = Math.floor(i * ratio_w);
                            var xx_stop = Math.ceil((i + 1) * ratio_w);
                            var yy_start = Math.floor(j * ratio_h);
                            var yy_stop = Math.ceil((j + 1) * ratio_h);

                            xx_stop = Math.min(xx_stop, width_source);
                            yy_stop = Math.min(yy_stop, height_source);

                            for (var yy = yy_start; yy < yy_stop; yy++) {
                                var dy = Math.abs(center_y - yy) / ratio_h_half;
                                var center_x = i * ratio_w;
                                var w0 = dy * dy; //pre-calc part of w
                                for (var xx = xx_start; xx < xx_stop; xx++) {
                                    var dx = Math.abs(center_x - xx) / ratio_w_half;
                                    var w = Math.sqrt(w0 + dx * dx);
                                    if (w >= 1) {
                                        //pixel too far
                                        continue;
                                    }
                                    //hermite filter
                                    weight = 2 * w * w * w - 3 * w * w + 1;
                                    //calc source pixel location
                                    var pos_x = 4 * (xx + yy * width_source);
                                    //alpha
                                    gx_a += weight * source[pos_x + 3];
                                    weights_alpha += weight;
                                    //colors
                                    if (source[pos_x + 3] < 255)
                                        weight = weight * source[pos_x + 3] / 250;
                                    gx_r += weight * source[pos_x];
                                    gx_g += weight * source[pos_x + 1];
                                    gx_b += weight * source[pos_x + 2];
                                    weights += weight;
                                }
                            }
                            target[x2] = gx_r / weights;
                            target[x2 + 1] = gx_g / weights;
                            target[x2 + 2] = gx_b / weights;
                            target[x2 + 3] = gx_a / weights_alpha;
                        }
                    }

                    //return
                    var objData = {
                        core: core,
                        target: target,
                    };
                    postMessage(objData, [target.buffer]);
                };
                //end worker
            }.toString(),
            ')()'], { type: 'application/javascript' }));

        /**
         * Hermite resize - fast image resize/resample using Hermite filter. 1 cpu version!
         * 
         * @param {HtmlElement} canvas
         * @param {int} width
         * @param {int} height
         * @param {boolean} resize_canvas if true, canvas will be resized. Optional.
         */
        this.resample_single = function (canvas, width, height, resize_canvas) {
            var width_source = canvas.width;
            var height_source = canvas.height;
            width = Math.round(width);
            height = Math.round(height);

            var ratio_w = width_source / width;
            var ratio_h = height_source / height;
            var ratio_w_half = Math.ceil(ratio_w / 2);
            var ratio_h_half = Math.ceil(ratio_h / 2);

            var ctx = canvas.getContext("2d");
            var img = ctx.getImageData(0, 0, width_source, height_source);
            var img2 = ctx.createImageData(width, height);
            var data = img.data;
            var data2 = img2.data;

            for (var j = 0; j < height; j++) {
                for (var i = 0; i < width; i++) {
                    var x2 = (i + j * width) * 4;
                    var weight = 0;
                    var weights = 0;
                    var weights_alpha = 0;
                    var gx_r = 0;
                    var gx_g = 0;
                    var gx_b = 0;
                    var gx_a = 0;
                    var center_y = j * ratio_h;

                    var xx_start = Math.floor(i * ratio_w);
                    var xx_stop = Math.ceil((i + 1) * ratio_w);
                    var yy_start = Math.floor(j * ratio_h);
                    var yy_stop = Math.ceil((j + 1) * ratio_h);
                    xx_stop = Math.min(xx_stop, width_source);
                    yy_stop = Math.min(yy_stop, height_source);

                    for (var yy = yy_start; yy < yy_stop; yy++) {
                        var dy = Math.abs(center_y - yy) / ratio_h_half;
                        var center_x = i * ratio_w;
                        var w0 = dy * dy; //pre-calc part of w
                        for (var xx = xx_start; xx < xx_stop; xx++) {
                            var dx = Math.abs(center_x - xx) / ratio_w_half;
                            var w = Math.sqrt(w0 + dx * dx);
                            if (w >= 1) {
                                //pixel too far
                                continue;
                            }
                            //hermite filter
                            weight = 2 * w * w * w - 3 * w * w + 1;
                            var pos_x = 4 * (xx + yy * width_source);
                            //alpha
                            gx_a += weight * data[pos_x + 3];
                            weights_alpha += weight;
                            //colors
                            if (data[pos_x + 3] < 255)
                                weight = weight * data[pos_x + 3] / 250;
                            gx_r += weight * data[pos_x];
                            gx_g += weight * data[pos_x + 1];
                            gx_b += weight * data[pos_x + 2];
                            weights += weight;
                        }
                    }
                    data2[x2] = gx_r / weights;
                    data2[x2 + 1] = gx_g / weights;
                    data2[x2 + 2] = gx_b / weights;
                    data2[x2 + 3] = gx_a / weights_alpha;
                }
            }
            //clear and resize canvas
            if (resize_canvas === true) {
                canvas.width = width;
                canvas.height = height;
            }
            else {
                ctx.clearRect(0, 0, width_source, height_source);
            }

            //draw
            ctx.putImageData(img2, 0, 0);
        };
    }
    window.HERMITE = new Hermite();
</script></head>


<body>
    <style>
        input {
            max-width: 100px;
        }

        .controls-form {
            padding-bottom: 1em
        }

        .hide {
            display: none !important;
        }

        #dimensions-form {
            display: inline;
        }

        #download-link {
            display: block;
        }

        .grid-wrapper {
            flex-flow: row wrap;
        }

        .container {
            flex: 1 1 50%;
            display: inline-block;
            margin-bottom: 2em;
        }

        .cr-viewport {
            border: 1px black solid !important;
        }

        .controls {
            display: block;
            padding-top: 1em;
            padding-bottom: 1em;
        }
    </style>
    <div class="demo-wrap upload-demo">
        <div class="controls-form">
            <form id="dimensions-form">
                <label for="width">Width</label>
                <input id="width" value="150">
                <label for="height">Height</label>
                <input id="height" value="150">
                <label for="count">Count</label>
                <input id="count" value="4">
                <label for="columns">Columns</label>
                <input id="columns" value="2">
                <button>Create</button>
            </form>
            <button id="reset-all">Reset</button>
        </div>
        <div class="grid-wrapper" style="width: 450px">
        <div class="container">
        <div class="upload-demo-wrap">
            <div id="upload-demo-0" style="height: 152px; width: 152px" class="croppie-container"><div class="cr-boundary" aria-dropeffect="none"><img class="cr-image" alt="preview" aria-grabbed="false"><div class="cr-viewport cr-vp-square" tabindex="0" style="width: 150px; height: 150px;"></div><div class="cr-overlay"></div></div><div class="cr-slider-wrap"><input class="cr-slider" type="range" step="0.0001" aria-label="zoom"></div></div>
        </div>
        <input type="file" id="upload-0" value="Choose a file" accept="image/*">
        
        </div><div class="container">
        <div class="upload-demo-wrap">
            <div id="upload-demo-1" style="height: 152px; width: 152px" class="croppie-container"><div class="cr-boundary" aria-dropeffect="none"><img class="cr-image" alt="preview" aria-grabbed="false"><div class="cr-viewport cr-vp-square" tabindex="0" style="width: 150px; height: 150px;"></div><div class="cr-overlay"></div></div><div class="cr-slider-wrap"><input class="cr-slider" type="range" step="0.0001" aria-label="zoom"></div></div>
        </div>
        <input type="file" id="upload-1" value="Choose a file" accept="image/*">
        
        </div><div class="container">
        <div class="upload-demo-wrap">
            <div id="upload-demo-2" style="height: 152px; width: 152px" class="croppie-container"><div class="cr-boundary" aria-dropeffect="none"><img class="cr-image" alt="preview" aria-grabbed="false"><div class="cr-viewport cr-vp-square" tabindex="0" style="width: 150px; height: 150px;"></div><div class="cr-overlay"></div></div><div class="cr-slider-wrap"><input class="cr-slider" type="range" step="0.0001" aria-label="zoom"></div></div>
        </div>
        <input type="file" id="upload-2" value="Choose a file" accept="image/*">
        
        </div><div class="container">
        <div class="upload-demo-wrap">
            <div id="upload-demo-3" style="height: 152px; width: 152px" class="croppie-container"><div class="cr-boundary" aria-dropeffect="none"><img class="cr-image" alt="preview" aria-grabbed="false"><div class="cr-viewport cr-vp-square" tabindex="0" style="width: 150px; height: 150px;"></div><div class="cr-overlay"></div></div><div class="cr-slider-wrap"><input class="cr-slider" type="range" step="0.0001" aria-label="zoom"></div></div>
        </div>
        <input type="file" id="upload-3" value="Choose a file" accept="image/*">
        
        </div></div>
    </div>
    <p></p>
    <section class="controls">
        <button id="final-render">Render</button>
        <label>Quality</label><input id="quality" value="1.0" type="number" step="0.01" min="0.1" max="1.0">
    </section>
    <a id="download-link" class="hide" href="#" target="_blank" download="export.jpg">Download</a>
    <img id="result">
    <canvas id="outcanvas" style="display: none">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/croppie/2.6.5/croppie.min.js"></script>
    <script>
        function readFile(input, cb) {
            if (input.files && input.files[0]) {
                var reader = new FileReader();

                reader.onload = cb;

                reader.readAsDataURL(input.files[0]);
            } else if (input.files.length === 0) {
                // Do nothing, user cancelled
            } else {
                alert("Sorry - you're browser doesn't support the FileReader API");
            }
        }

        function createSingleImageBox(n, width, height) {
            var container = document.createElement('div');
            container.classList.add('container');
            var style = `height: ${height + 2}px; width: ${width + 2}px`;

            container.innerHTML = `
        <div class="upload-demo-wrap">
            <div id="upload-demo-${n}" style="${style}"></div>
        </div>
        <input type="file" id="upload-${n}" value="Choose a file" accept="image/*" />
        </div>
        `;
            document.querySelector('.grid-wrapper').appendChild(container);

            var croppie = new Croppie(document.getElementById(`upload-demo-${n}`), {
                viewport: {
                    width,
                    height,
                    type: 'square'
                },
            });
            document.getElementById(`upload-${n}`).onchange = function () {
                readFile(this, function (e) {
                    croppie.bind({
                        url: e.target.result
                    });
                });
            };
            return croppie;
        }

        (function init() {
            var croppies = [];
            var height, width;
            document.getElementById('dimensions-form').onsubmit = function (e) {
                e.preventDefault();
                height = parseInt(document.getElementById('height').value, 10);
                width = parseInt(document.getElementById('width').value, 10);
                var count = parseInt(document.getElementById('count').value, 10);
                var columns = parseInt(document.getElementById('columns').value, 10);
                var gridWrapper = document.querySelector('.grid-wrapper');
                gridWrapper.setAttribute('style', `width: ${width * (columns + 1)}px`);
                for (var i = 0; i < count; i++) {
                    if (!croppies[i]) {
                        croppies.push(createSingleImageBox(i, width, height));
                    }
                }
                croppies = croppies.slice(0, count);
                var length = gridWrapper.children.length;
                for (var j = count; j < length; j++) {
                    gridWrapper.children[gridWrapper.children.length - 1].remove();
                }
            };
            document.getElementById('reset-all').onclick = function reset() {
                document.querySelector('.grid-wrapper').innerHTML = '';
                croppies = [];
                var img = document.getElementById('result');
                img.classList.add('hide')
                document.querySelector('#download-link').classList.add('hide');
            };
            document.getElementById('final-render').onclick = function render() {
                var c = document.getElementById("outcanvas");
                c.width = width;
                c.height = height * croppies.length;
                var ctx = c.getContext("2d");
                ctx.fillStyle = 'white';
                ctx.fillRect(0, 0, c.width, c.height);
                for (var i = 0; i < croppies.length; i++) {
                    var dy = i * height;
                    var croppie = croppies[i];
                    if (!croppie.data.url) {
                        continue;
                    }
                    var originalWidth = croppie._originalImageWidth
                    var originalHeight = croppie._originalImageHeight;
                    var croppieData = croppie.get();
                    var zoom = croppieData.zoom;
                    var points = croppieData.points;
                    var temp_canvas = document.createElement("canvas");
                    temp_canvas.width = originalWidth;
                    temp_canvas.height = originalHeight;
                    var temp_ctx = temp_canvas.getContext("2d");
                    temp_ctx.drawImage(croppie.elements.img, 0, 0);

                    tmpwidth = originalWidth * zoom;
                    tmpheight = originalHeight * zoom;
                    tmpwidth = Math.round(tmpwidth);
                    tmpheight = Math.round(tmpheight);
                    HERMITE.resample_single(temp_canvas, tmpwidth, tmpheight, true);

                    ctx.drawImage(temp_canvas, Math.floor(points[0] * zoom), Math.floor(points[1] * zoom), width, height, 0, dy, width, height);
                }
                function exportImg() {
                    var c = document.getElementById("outcanvas");
                    var quality = parseFloat(document.getElementById('quality').value);
                    var img = c.toDataURL("image/jpeg", quality);
                    var imgnode = document.getElementById('result');
                    imgnode.classList.remove('hide');
                    imgnode.src = img;
                    imgnode.width = width;
                    imgnode.height = height * croppies.length;

                    var link = document.getElementById("download-link");
                    link.classList.remove('hide');
                    link.href = img;
                }
                exportImg();
            }
        })();

    </script>


</canvas><div class="jso-cursor-trail-wrapper" style="position: fixed; left: 0px; top: 0px; width: 100vw; height: 100vh; overflow: hidden; pointer-events: none; z-index: 9999;"><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 336 336">
              <path fill="#ffe940" d="M168 260.4C128.4 260.4 88.8 269.2 49.2 286.8 66.8 247.2 75.6 207.6 75.6 168 75.6 128.4 66.8 88.8 49.2 49.2 88.8 66.8 128.4 75.6 168 75.6 207.6 75.6 247.2 66.8 286.8 49.2 269.2 88.8 260.4 128.4 260.4 168 260.4 207.6 269.2 247.2 286.8 286.8 247.2 269.2 207.6 260.4 168 260.4Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="14.175824175824175" viewBox="0 0 91 86"><polygon points="45.5 71.3 17.6 85.9 22.9 54.8 0.3 32.8 31.5 28.3 45.5 0 59.5 28.3 90.7 32.8 68.1 54.8 73.4 85.9" fill="#1ab3c1"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 120 120"><defs><filter id="a" width="255.6%" height="255.6%" x="-77.8%" y="-77.8%" filterUnits="objectBoundingBox"><feGaussianBlur in="SourceGraphic" stdDeviation="14"></feGaussianBlur></filter></defs><circle cx="60" cy="60" r="27" fill="#7bbfd4" filter="url(#a)"></circle></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 120 120"><defs><filter id="a" width="255.6%" height="255.6%" x="-77.8%" y="-77.8%" filterUnits="objectBoundingBox"><feGaussianBlur in="SourceGraphic" stdDeviation="14"></feGaussianBlur></filter></defs><circle cx="60" cy="60" r="27" fill="#7bbfd4" filter="url(#a)"></circle></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="14.175824175824175" viewBox="0 0 91 86"><polygon points="45.5 71.3 17.6 85.9 22.9 54.8 0.3 32.8 31.5 28.3 45.5 0 59.5 28.3 90.7 32.8 68.1 54.8 73.4 85.9" fill="#f8ffc4"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="14.175824175824175" viewBox="0 0 91 86"><polygon points="45.5 71.3 17.6 85.9 22.9 54.8 0.3 32.8 31.5 28.3 45.5 0 59.5 28.3 90.7 32.8 68.1 54.8 73.4 85.9" fill="#1ab3c1"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#ffcf5f"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="18.03921568627451" viewBox="0 0 51 46"><polyline points="22.7 25.3 25.8 25.3 26 25.8 26.2 26.3 26.2 26.8 26.2 27.4 26.1 27.9 25.9 28.5 25.5 29.1 25.1 29.6 24.6 30.1 24 30.6 23.3 30.9 22.6 31.2 21.7 31.3 20.9 31.4 20 31.3 19.1 31.1 18.2 30.7 17.4 30.2 16.6 29.6 15.8 28.8 15.2 28 14.7 27 14.3 25.9 14.1 24.8 14 23.6 14.1 22.4 14.3 21.2 14.8 20 15.4 18.8 16.2 17.7 17.1 16.7 18.2 15.8 19.5 15.1 20.8 14.5 22.3 14.1 23.8 13.9 25.4 14 27 14.2 28.6 14.7 30.1 15.3 31.5 16.2 32.9 17.3 34 18.6 35.1 20.1 35.9 21.7 36.5 23.4 36.9 25.2 37 27.1 36.9 29 36.5 31 35.8 32.8 34.8 34.6 33.6 36.3 32.2 37.9 30.6 39.2 28.7 40.3 26.7 41.2 24.6 41.8 22.3 42.1 20.1 42.2 17.8 41.9 15.5 41.3 13.3 40.4 11.2 39.2 9.3 37.7 7.5 35.9 6 34 4.8 31.7 3.9 29.4 3.3 26.9 3 24.3 3.1 21.6 3.5 19 4.4 16.4 5.5 13.9 7 11.6 8.9 9.5 11 7.6 13.4 6 16 4.7 18.8 3.7 21.7 3.2 24.7 3 27.7 3.2 30.7 3.9 33.7 4.9 36.4 6.3 39 8.1 41.4 10.3 43.4 12.8 45.1 15.5 46.5 18.5 47.4 21.7 47.9 24.9 48 28.3 47.6 31.7 46.8 35 45.4 38.2 43.7 41.2 41.6 44" fill="none" stroke-width="5" stroke="#ffee96"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 336 336">
              <path fill="#ffe940" d="M168 260.4C128.4 260.4 88.8 269.2 49.2 286.8 66.8 247.2 75.6 207.6 75.6 168 75.6 128.4 66.8 88.8 49.2 49.2 88.8 66.8 128.4 75.6 168 75.6 207.6 75.6 247.2 66.8 286.8 49.2 269.2 88.8 260.4 128.4 260.4 168 260.4 207.6 269.2 247.2 286.8 286.8 247.2 269.2 207.6 260.4 168 260.4Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#ffcf5f"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#ffcf5f"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 336 336">
              <path fill="#ffe940" d="M168 260.4C128.4 260.4 88.8 269.2 49.2 286.8 66.8 247.2 75.6 207.6 75.6 168 75.6 128.4 66.8 88.8 49.2 49.2 88.8 66.8 128.4 75.6 168 75.6 207.6 75.6 247.2 66.8 286.8 49.2 269.2 88.8 260.4 128.4 260.4 168 260.4 207.6 269.2 247.2 286.8 286.8 247.2 269.2 207.6 260.4 168 260.4Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#ffcf5f"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 120 120"><defs><filter id="a" width="255.6%" height="255.6%" x="-77.8%" y="-77.8%" filterUnits="objectBoundingBox"><feGaussianBlur in="SourceGraphic" stdDeviation="14"></feGaussianBlur></filter></defs><circle cx="60" cy="60" r="27" fill="#7bbfd4" filter="url(#a)"></circle></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 782px; top: 422px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="30" height="30" viewBox="0 0 336 336">
              <path fill="#ffed8a" d="M168 235.2C128.4 235.2 88.8 252.4 49.2 286.8 83.6 247.2 100.8 207.6 100.8 168 100.8 128.4 83.6 88.8 49.2 49.2 88.8 83.6 128.4 100.8 168 100.8 207.6 100.8 247.2 83.6 286.8 49.2 252.4 88.8 235.2 128.4 235.2 168 235.2 207.6 252.4 247.2 286.8 286.8 247.2 252.4 207.6 235.2 168 235.2Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 781px; top: 438px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#fff"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 779px; top: 455px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 777px; top: 464px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 775px; top: 470px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 773px; top: 473px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 769px; top: 478px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="8" height="8" viewBox="0 0 142 142">
              <circle cx="71" cy="71" r="71" fill="#fff2a8"></circle>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 763px; top: 486px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 757px; top: 497px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#ffcf5f"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 753px; top: 506px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 753px; top: 512px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 753px; top: 522px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 759px; top: 549px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="18.03921568627451" viewBox="0 0 51 46"><polyline points="22.7 25.3 25.8 25.3 26 25.8 26.2 26.3 26.2 26.8 26.2 27.4 26.1 27.9 25.9 28.5 25.5 29.1 25.1 29.6 24.6 30.1 24 30.6 23.3 30.9 22.6 31.2 21.7 31.3 20.9 31.4 20 31.3 19.1 31.1 18.2 30.7 17.4 30.2 16.6 29.6 15.8 28.8 15.2 28 14.7 27 14.3 25.9 14.1 24.8 14 23.6 14.1 22.4 14.3 21.2 14.8 20 15.4 18.8 16.2 17.7 17.1 16.7 18.2 15.8 19.5 15.1 20.8 14.5 22.3 14.1 23.8 13.9 25.4 14 27 14.2 28.6 14.7 30.1 15.3 31.5 16.2 32.9 17.3 34 18.6 35.1 20.1 35.9 21.7 36.5 23.4 36.9 25.2 37 27.1 36.9 29 36.5 31 35.8 32.8 34.8 34.6 33.6 36.3 32.2 37.9 30.6 39.2 28.7 40.3 26.7 41.2 24.6 41.8 22.3 42.1 20.1 42.2 17.8 41.9 15.5 41.3 13.3 40.4 11.2 39.2 9.3 37.7 7.5 35.9 6 34 4.8 31.7 3.9 29.4 3.3 26.9 3 24.3 3.1 21.6 3.5 19 4.4 16.4 5.5 13.9 7 11.6 8.9 9.5 11 7.6 13.4 6 16 4.7 18.8 3.7 21.7 3.2 24.7 3 27.7 3.2 30.7 3.9 33.7 4.9 36.4 6.3 39 8.1 41.4 10.3 43.4 12.8 45.1 15.5 46.5 18.5 47.4 21.7 47.9 24.9 48 28.3 47.6 31.7 46.8 35 45.4 38.2 43.7 41.2 41.6 44" fill="none" stroke-width="5" stroke="#ffee96"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 761px; top: 561px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="18.03921568627451" viewBox="0 0 51 46"><polyline points="22.7 25.3 25.8 25.3 26 25.8 26.2 26.3 26.2 26.8 26.2 27.4 26.1 27.9 25.9 28.5 25.5 29.1 25.1 29.6 24.6 30.1 24 30.6 23.3 30.9 22.6 31.2 21.7 31.3 20.9 31.4 20 31.3 19.1 31.1 18.2 30.7 17.4 30.2 16.6 29.6 15.8 28.8 15.2 28 14.7 27 14.3 25.9 14.1 24.8 14 23.6 14.1 22.4 14.3 21.2 14.8 20 15.4 18.8 16.2 17.7 17.1 16.7 18.2 15.8 19.5 15.1 20.8 14.5 22.3 14.1 23.8 13.9 25.4 14 27 14.2 28.6 14.7 30.1 15.3 31.5 16.2 32.9 17.3 34 18.6 35.1 20.1 35.9 21.7 36.5 23.4 36.9 25.2 37 27.1 36.9 29 36.5 31 35.8 32.8 34.8 34.6 33.6 36.3 32.2 37.9 30.6 39.2 28.7 40.3 26.7 41.2 24.6 41.8 22.3 42.1 20.1 42.2 17.8 41.9 15.5 41.3 13.3 40.4 11.2 39.2 9.3 37.7 7.5 35.9 6 34 4.8 31.7 3.9 29.4 3.3 26.9 3 24.3 3.1 21.6 3.5 19 4.4 16.4 5.5 13.9 7 11.6 8.9 9.5 11 7.6 13.4 6 16 4.7 18.8 3.7 21.7 3.2 24.7 3 27.7 3.2 30.7 3.9 33.7 4.9 36.4 6.3 39 8.1 41.4 10.3 43.4 12.8 45.1 15.5 46.5 18.5 47.4 21.7 47.9 24.9 48 28.3 47.6 31.7 46.8 35 45.4 38.2 43.7 41.2 41.6 44" fill="none" stroke-width="5" stroke="#ffee96"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 761px; top: 566px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="14.175824175824175" viewBox="0 0 91 86"><polygon points="45.5 71.3 17.6 85.9 22.9 54.8 0.3 32.8 31.5 28.3 45.5 0 59.5 28.3 90.7 32.8 68.1 54.8 73.4 85.9" fill="#1ab3c1"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 761px; top: 571px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="8" height="8" viewBox="0 0 142 142">
              <circle cx="71" cy="71" r="71" fill="#fff2a8"></circle>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 759px; top: 573px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 711px; top: 129px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 716px; top: 87px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 718px; top: 33px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 336 336">
              <path fill="#ffe940" d="M168 260.4C128.4 260.4 88.8 269.2 49.2 286.8 66.8 247.2 75.6 207.6 75.6 168 75.6 128.4 66.8 88.8 49.2 49.2 88.8 66.8 128.4 75.6 168 75.6 207.6 75.6 247.2 66.8 286.8 49.2 269.2 88.8 260.4 128.4 260.4 168 260.4 207.6 269.2 247.2 286.8 286.8 247.2 269.2 207.6 260.4 168 260.4Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 718px; top: 30px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#fff"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 695px; top: 35px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 336 336">
              <path fill="#ffe940" d="M168 260.4C128.4 260.4 88.8 269.2 49.2 286.8 66.8 247.2 75.6 207.6 75.6 168 75.6 128.4 66.8 88.8 49.2 49.2 88.8 66.8 128.4 75.6 168 75.6 207.6 75.6 247.2 66.8 286.8 49.2 269.2 88.8 260.4 128.4 260.4 168 260.4 207.6 269.2 247.2 286.8 286.8 247.2 269.2 207.6 260.4 168 260.4Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 680px; top: 36px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 336 336">
              <path fill="#ffe940" d="M168 260.4C128.4 260.4 88.8 269.2 49.2 286.8 66.8 247.2 75.6 207.6 75.6 168 75.6 128.4 66.8 88.8 49.2 49.2 88.8 66.8 128.4 75.6 168 75.6 207.6 75.6 247.2 66.8 286.8 49.2 269.2 88.8 260.4 128.4 260.4 168 260.4 207.6 269.2 247.2 286.8 286.8 247.2 269.2 207.6 260.4 168 260.4Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 637px; top: 29px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="30" height="30" viewBox="0 0 336 336">
              <path fill="#ffed8a" d="M168 235.2C128.4 235.2 88.8 252.4 49.2 286.8 83.6 247.2 100.8 207.6 100.8 168 100.8 128.4 83.6 88.8 49.2 49.2 88.8 83.6 128.4 100.8 168 100.8 207.6 100.8 247.2 83.6 286.8 49.2 252.4 88.8 235.2 128.4 235.2 168 235.2 207.6 252.4 247.2 286.8 286.8 247.2 252.4 207.6 235.2 168 235.2Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 566px; top: 3px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="8" height="8" viewBox="0 0 142 142">
              <circle cx="71" cy="71" r="71" fill="#fff2a8"></circle>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="30" height="30" viewBox="0 0 336 336">
              <path fill="#ffed8a" d="M168 235.2C128.4 235.2 88.8 252.4 49.2 286.8 83.6 247.2 100.8 207.6 100.8 168 100.8 128.4 83.6 88.8 49.2 49.2 88.8 83.6 128.4 100.8 168 100.8 207.6 100.8 247.2 83.6 286.8 49.2 252.4 88.8 235.2 128.4 235.2 168 235.2 207.6 252.4 247.2 286.8 286.8 247.2 252.4 207.6 235.2 168 235.2Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="30" height="30" viewBox="0 0 336 336">
              <path fill="#ffed8a" d="M168 235.2C128.4 235.2 88.8 252.4 49.2 286.8 83.6 247.2 100.8 207.6 100.8 168 100.8 128.4 83.6 88.8 49.2 49.2 88.8 83.6 128.4 100.8 168 100.8 207.6 100.8 247.2 83.6 286.8 49.2 252.4 88.8 235.2 128.4 235.2 168 235.2 207.6 252.4 247.2 286.8 286.8 247.2 252.4 207.6 235.2 168 235.2Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="14.175824175824175" viewBox="0 0 91 86"><polygon points="45.5 71.3 17.6 85.9 22.9 54.8 0.3 32.8 31.5 28.3 45.5 0 59.5 28.3 90.7 32.8 68.1 54.8 73.4 85.9" fill="#f8ffc4"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="14.175824175824175" viewBox="0 0 91 86"><polygon points="45.5 71.3 17.6 85.9 22.9 54.8 0.3 32.8 31.5 28.3 45.5 0 59.5 28.3 90.7 32.8 68.1 54.8 73.4 85.9" fill="#1ab3c1"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 120 120"><defs><filter id="a" width="255.6%" height="255.6%" x="-77.8%" y="-77.8%" filterUnits="objectBoundingBox"><feGaussianBlur in="SourceGraphic" stdDeviation="14"></feGaussianBlur></filter></defs><circle cx="60" cy="60" r="27" fill="#7bbfd4" filter="url(#a)"></circle></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="18.03921568627451" viewBox="0 0 51 46"><polyline points="22.7 25.3 25.8 25.3 26 25.8 26.2 26.3 26.2 26.8 26.2 27.4 26.1 27.9 25.9 28.5 25.5 29.1 25.1 29.6 24.6 30.1 24 30.6 23.3 30.9 22.6 31.2 21.7 31.3 20.9 31.4 20 31.3 19.1 31.1 18.2 30.7 17.4 30.2 16.6 29.6 15.8 28.8 15.2 28 14.7 27 14.3 25.9 14.1 24.8 14 23.6 14.1 22.4 14.3 21.2 14.8 20 15.4 18.8 16.2 17.7 17.1 16.7 18.2 15.8 19.5 15.1 20.8 14.5 22.3 14.1 23.8 13.9 25.4 14 27 14.2 28.6 14.7 30.1 15.3 31.5 16.2 32.9 17.3 34 18.6 35.1 20.1 35.9 21.7 36.5 23.4 36.9 25.2 37 27.1 36.9 29 36.5 31 35.8 32.8 34.8 34.6 33.6 36.3 32.2 37.9 30.6 39.2 28.7 40.3 26.7 41.2 24.6 41.8 22.3 42.1 20.1 42.2 17.8 41.9 15.5 41.3 13.3 40.4 11.2 39.2 9.3 37.7 7.5 35.9 6 34 4.8 31.7 3.9 29.4 3.3 26.9 3 24.3 3.1 21.6 3.5 19 4.4 16.4 5.5 13.9 7 11.6 8.9 9.5 11 7.6 13.4 6 16 4.7 18.8 3.7 21.7 3.2 24.7 3 27.7 3.2 30.7 3.9 33.7 4.9 36.4 6.3 39 8.1 41.4 10.3 43.4 12.8 45.1 15.5 46.5 18.5 47.4 21.7 47.9 24.9 48 28.3 47.6 31.7 46.8 35 45.4 38.2 43.7 41.2 41.6 44" fill="none" stroke-width="5" stroke="#ffee96"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="30" height="30" viewBox="0 0 336 336">
              <path fill="#ffed8a" d="M168 235.2C128.4 235.2 88.8 252.4 49.2 286.8 83.6 247.2 100.8 207.6 100.8 168 100.8 128.4 83.6 88.8 49.2 49.2 88.8 83.6 128.4 100.8 168 100.8 207.6 100.8 247.2 83.6 286.8 49.2 252.4 88.8 235.2 128.4 235.2 168 235.2 207.6 252.4 247.2 286.8 286.8 247.2 252.4 207.6 235.2 168 235.2Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="30" height="30" viewBox="0 0 336 336">
              <path fill="#ffed8a" d="M168 235.2C128.4 235.2 88.8 252.4 49.2 286.8 83.6 247.2 100.8 207.6 100.8 168 100.8 128.4 83.6 88.8 49.2 49.2 88.8 83.6 128.4 100.8 168 100.8 207.6 100.8 247.2 83.6 286.8 49.2 252.4 88.8 235.2 128.4 235.2 168 235.2 207.6 252.4 247.2 286.8 286.8 247.2 252.4 207.6 235.2 168 235.2Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="8" height="8" viewBox="0 0 142 142">
              <circle cx="71" cy="71" r="71" fill="#fff2a8"></circle>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 120 120"><defs><filter id="a" width="255.6%" height="255.6%" x="-77.8%" y="-77.8%" filterUnits="objectBoundingBox"><feGaussianBlur in="SourceGraphic" stdDeviation="14"></feGaussianBlur></filter></defs><circle cx="60" cy="60" r="27" fill="#7bbfd4" filter="url(#a)"></circle></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="18.03921568627451" viewBox="0 0 51 46"><polyline points="22.7 25.3 25.8 25.3 26 25.8 26.2 26.3 26.2 26.8 26.2 27.4 26.1 27.9 25.9 28.5 25.5 29.1 25.1 29.6 24.6 30.1 24 30.6 23.3 30.9 22.6 31.2 21.7 31.3 20.9 31.4 20 31.3 19.1 31.1 18.2 30.7 17.4 30.2 16.6 29.6 15.8 28.8 15.2 28 14.7 27 14.3 25.9 14.1 24.8 14 23.6 14.1 22.4 14.3 21.2 14.8 20 15.4 18.8 16.2 17.7 17.1 16.7 18.2 15.8 19.5 15.1 20.8 14.5 22.3 14.1 23.8 13.9 25.4 14 27 14.2 28.6 14.7 30.1 15.3 31.5 16.2 32.9 17.3 34 18.6 35.1 20.1 35.9 21.7 36.5 23.4 36.9 25.2 37 27.1 36.9 29 36.5 31 35.8 32.8 34.8 34.6 33.6 36.3 32.2 37.9 30.6 39.2 28.7 40.3 26.7 41.2 24.6 41.8 22.3 42.1 20.1 42.2 17.8 41.9 15.5 41.3 13.3 40.4 11.2 39.2 9.3 37.7 7.5 35.9 6 34 4.8 31.7 3.9 29.4 3.3 26.9 3 24.3 3.1 21.6 3.5 19 4.4 16.4 5.5 13.9 7 11.6 8.9 9.5 11 7.6 13.4 6 16 4.7 18.8 3.7 21.7 3.2 24.7 3 27.7 3.2 30.7 3.9 33.7 4.9 36.4 6.3 39 8.1 41.4 10.3 43.4 12.8 45.1 15.5 46.5 18.5 47.4 21.7 47.9 24.9 48 28.3 47.6 31.7 46.8 35 45.4 38.2 43.7 41.2 41.6 44" fill="none" stroke-width="5" stroke="#ffee96"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="30" height="30" viewBox="0 0 336 336">
              <path fill="#ffed8a" d="M168 235.2C128.4 235.2 88.8 252.4 49.2 286.8 83.6 247.2 100.8 207.6 100.8 168 100.8 128.4 83.6 88.8 49.2 49.2 88.8 83.6 128.4 100.8 168 100.8 207.6 100.8 247.2 83.6 286.8 49.2 252.4 88.8 235.2 128.4 235.2 168 235.2 207.6 252.4 247.2 286.8 286.8 247.2 252.4 207.6 235.2 168 235.2Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="14.175824175824175" viewBox="0 0 91 86"><polygon points="45.5 71.3 17.6 85.9 22.9 54.8 0.3 32.8 31.5 28.3 45.5 0 59.5 28.3 90.7 32.8 68.1 54.8 73.4 85.9" fill="#f8ffc4"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 0px; top: 0px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#ffcf5f"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 782px; top: 422px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 781px; top: 438px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 779px; top: 455px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 777px; top: 464px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#fff"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 775px; top: 470px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 773px; top: 473px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 769px; top: 478px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 763px; top: 486px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 757px; top: 497px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#fff"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 753px; top: 506px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 336 336">
              <path fill="#ffe940" d="M168 260.4C128.4 260.4 88.8 269.2 49.2 286.8 66.8 247.2 75.6 207.6 75.6 168 75.6 128.4 66.8 88.8 49.2 49.2 88.8 66.8 128.4 75.6 168 75.6 207.6 75.6 247.2 66.8 286.8 49.2 269.2 88.8 260.4 128.4 260.4 168 260.4 207.6 269.2 247.2 286.8 286.8 247.2 269.2 207.6 260.4 168 260.4Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 753px; top: 512px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="30" height="30" viewBox="0 0 336 336">
              <path fill="#ffed8a" d="M168 235.2C128.4 235.2 88.8 252.4 49.2 286.8 83.6 247.2 100.8 207.6 100.8 168 100.8 128.4 83.6 88.8 49.2 49.2 88.8 83.6 128.4 100.8 168 100.8 207.6 100.8 247.2 83.6 286.8 49.2 252.4 88.8 235.2 128.4 235.2 168 235.2 207.6 252.4 247.2 286.8 286.8 247.2 252.4 207.6 235.2 168 235.2Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 753px; top: 522px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 759px; top: 549px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 761px; top: 561px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="17" height="17.184782608695652" viewBox="0 0 92 93"><polygon points="47.5 71.3 26.9 90.3 28.9 62.3 1.2 58.1 24.3 42.2 10.4 17.9 37.2 26.1 47.5 0 57.8 26.1 84.6 17.9 70.7 42.2 93.8 58.1 66.1 62.3 68.1 90.3" transform="rotate(8 47.5 47.5)" fill="#ffed8a"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 761px; top: 566px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 761px; top: 571px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#ffcf5f"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 759px; top: 573px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="19.696969696969695" viewBox="0 0 66 65"><polyline points="63 1 62.9 2 62.7 2.9 62.4 3.9 62.1 4.8 61.7 5.8 61.2 6.7 60.7 7.6 60 8.5 59.4 9.4 58.6 10.3 57.8 11.1 57 11.9 56.1 12.7 55.1 13.5 54.1 14.2 53 14.9 51.9 15.6 50.7 16.3 49.5 16.9 48.3 17.5 47 18.1 45.7 18.7 44.4 19.2 43 19.6 41.6 20.1 40.2 20.5 38.8 20.8 37.4 21.2 36 21.5 34.5 21.8 33.1 22 31.7 22.2 30.2 22.3 28.8 22.5 27.4 22.6 26 22.6 24.6 22.7 23.3 22.7 21.9 22.6 20.6 22.6 19.3 22.5 18.1 22.3 16.9 22.2 15.7 22 14.5 21.8 13.5 21.5 12.4 21.3 11.4 21 10.4 20.7 9.5 20.4 8.7 20 7.9 19.7 7.2 19.3 6.5 18.9 5.9 18.5 5.3 18 4.8 17.6 4.3 17.2 4 16.7 3.6 16.2 3.4 15.8 3.2 15.3 3.1 14.8 3 14.4 3 13.9 3.1 13.4 3.2 13 3.4 12.5 3.6 12.1 3.9 11.6 4.3 11.2 4.7 10.8 5.2 10.4 5.7 10 6.3 9.6 6.9 9.2 7.6 8.9 8.3 8.6 9.1 8.3 9.9 8 10.8 7.7 11.7 7.5 12.6 7.2 13.6 7.1 14.6 6.9 15.7 6.7 16.7 6.6 17.8 6.5 18.9 6.5 20 6.4 21.2 6.4 22.3 6.5 23.5 6.5 24.7 6.6 25.9 6.7 27.1 6.8 28.2 7 29.4 7.2 30.6 7.4 31.8 7.7 32.9 8 34.1 8.3 35.2 8.6 36.3 9 37.4 9.3 38.4 9.7 39.5 10.2 40.5 10.6 41.5 11.1 42.4 11.6 43.3 12.2 44.2 12.7 45 13.3 45.8 13.9 46.6 14.5 47.3 15.1 48 15.7 48.6 16.4 49.2 17 49.7 17.7 50.2 18.4 50.6 19.1 51 19.8 51.4 20.5 51.7 21.2 51.9 21.9 52.1 22.6 52.2 23.3 52.3 24.1 52.3 24.8 52.3 25.5 52.2 26.2 52.1 26.9 51.9 27.6 51.7 28.3 51.4 29 51.1 29.7 50.8 30.4 50.4 31 49.9 31.7 49.4 32.3 48.9 33 48.4 33.6 47.7 34.2 47.1 34.7 46.4 35.3 45.7 35.8 45 36.4 44.2 36.9 43.5 37.3 42.6 37.8 41.8 38.3 41 38.7 40.1 39.1 39.2 39.5 38.3 39.8 37.4 40.1 36.5 40.4 35.6 40.7 34.6 41 33.7 41.2 32.8 41.5 31.8 41.7 30.9 41.8 30 42 29.1 42.1 28.2 42.2 27.3 42.3 26.4 42.4 25.6 42.4 24.7 42.4 23.9 42.5 23.1 42.4 22.4 42.4 21.6 42.4 20.9 42.3 20.2 42.2 19.5 42.1 18.9 42 18.3 41.9 17.7 41.8 17.2 41.6 16.7 41.5 16.3 41.3 15.8 41.1 15.5 40.9 15.1 40.7 14.8 40.5 14.5 40.3 14.3 40.1 14.1 39.9 13.9 39.7 13.8 39.5 13.8 39.3 13.7 39 13.7 38.8 13.8 38.6 13.8 38.4 14 38.2 14.1 38 14.3 37.8 14.5 37.6 14.8 37.5 15.1 37.3 15.4 37.1 15.8 37 16.1 36.8 16.6 36.7 17 36.6 17.5 36.5 18 36.4 18.5 36.3 19 36.3 19.6 36.2 20.2 36.2 20.8 36.2 21.4 36.2 22 36.2 22.6 36.2 23.3 36.3 23.9 36.3 24.6 36.4 25.3 36.5 26 36.6 26.6 36.8 27.3 36.9 28 37.1 28.7 37.3 29.3 37.5 30 37.7 30.7 37.9 31.3 38.2 32 38.4 32.6 38.7 33.2 39 33.8 39.3 34.4 39.6 35 39.9 35.5 40.3 36.1 40.7 36.6 41 37.1 41.4 37.5 41.8 38 42.2 38.4 42.6 38.8 43 39.2 43.5 39.5 43.9 39.9 44.3 40.2 44.8 40.4 45.2 40.7 45.7 40.9 46.2 41.1 46.6 41.2 47.1 41.4 47.5 41.5 48 41.5 48.5 41.6 49 41.6 49.4 41.6 49.9 41.6 50.4 41.5 50.8 41.4 51.3 41.3 51.7 41.2 52.2 41 52.6 40.9 53.1 40.7 53.5 40.4 53.9 40.2 54.3 39.9 54.7 39.6 55.1 39.3 55.5 39 55.9 38.7 56.3 38.3 56.6 38 57 37.6 57.3 37.2 57.7 36.8 58 36.4 58.3 36 58.6 35.6 58.9 35.2 59.2 34.7 59.4 34.3 59.7 33.9 59.9 33.4 60.1 33 60.4 32.6 60.6 32.1 60.8 31.7 61 31.3 61.1 30.9 61.3 30.5 61.5 30.1 61.6 29.7 61.8 29.3 61.9 28.9 62" fill="none" stroke-width="5" stroke="#f1b349"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 711px; top: 129px; pointer-events: none; display: none;">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="18.03921568627451" viewBox="0 0 51 46"><polyline points="22.7 25.3 25.8 25.3 26 25.8 26.2 26.3 26.2 26.8 26.2 27.4 26.1 27.9 25.9 28.5 25.5 29.1 25.1 29.6 24.6 30.1 24 30.6 23.3 30.9 22.6 31.2 21.7 31.3 20.9 31.4 20 31.3 19.1 31.1 18.2 30.7 17.4 30.2 16.6 29.6 15.8 28.8 15.2 28 14.7 27 14.3 25.9 14.1 24.8 14 23.6 14.1 22.4 14.3 21.2 14.8 20 15.4 18.8 16.2 17.7 17.1 16.7 18.2 15.8 19.5 15.1 20.8 14.5 22.3 14.1 23.8 13.9 25.4 14 27 14.2 28.6 14.7 30.1 15.3 31.5 16.2 32.9 17.3 34 18.6 35.1 20.1 35.9 21.7 36.5 23.4 36.9 25.2 37 27.1 36.9 29 36.5 31 35.8 32.8 34.8 34.6 33.6 36.3 32.2 37.9 30.6 39.2 28.7 40.3 26.7 41.2 24.6 41.8 22.3 42.1 20.1 42.2 17.8 41.9 15.5 41.3 13.3 40.4 11.2 39.2 9.3 37.7 7.5 35.9 6 34 4.8 31.7 3.9 29.4 3.3 26.9 3 24.3 3.1 21.6 3.5 19 4.4 16.4 5.5 13.9 7 11.6 8.9 9.5 11 7.6 13.4 6 16 4.7 18.8 3.7 21.7 3.2 24.7 3 27.7 3.2 30.7 3.9 33.7 4.9 36.4 6.3 39 8.1 41.4 10.3 43.4 12.8 45.1 15.5 46.5 18.5 47.4 21.7 47.9 24.9 48 28.3 47.6 31.7 46.8 35 45.4 38.2 43.7 41.2 41.6 44" fill="none" stroke-width="5" stroke="#ffee96"></polyline></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 716px; top: 87px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="14.175824175824175" viewBox="0 0 91 86"><polygon points="45.5 71.3 17.6 85.9 22.9 54.8 0.3 32.8 31.5 28.3 45.5 0 59.5 28.3 90.7 32.8 68.1 54.8 73.4 85.9" fill="#f8ffc4"></polygon></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 718px; top: 33px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="20" height="25.31645569620253" viewBox="0 0 158 200"><g fill="#fffb84"><path d="M81.414 5.788a87.588 87.588 0 0113.97-4.074c-35.942 21.191-52.388 65.835-36.94 106.08 15.45 40.246 57.544 62.418 98.432 54.117a87.569 87.569 0 01-13.106 6.32c-44.857 17.219-95.18-5.186-112.4-50.044-17.218-44.857 5.187-95.18 50.044-112.4z"></path><path d="M106.45 61.888c-4.714-4.714-11.476-7.381-20.285-8 8.81-.62 15.57-3.286 20.285-8 4.714-4.714 7.38-11.476 8-20.285.619 8.81 3.286 15.57 8 20.285 4.714 4.714 11.475 7.38 20.284 8-8.809.619-15.57 3.286-20.284 8-4.714 4.714-7.381 11.475-8 20.284-.62-8.809-3.286-15.57-8-20.284zM125.914 118.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214zM86.914 105.352c-3.536-3.535-8.607-5.535-15.214-6 6.607-.464 11.678-2.464 15.214-6 3.535-3.535 5.535-8.606 6-15.213.464 6.607 2.464 11.678 6 15.213 3.535 3.536 8.606 5.536 15.213 6-6.607.465-11.678 2.465-15.213 6-3.536 3.536-5.536 8.607-6 15.214-.465-6.607-2.465-11.678-6-15.214z"></path></g></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 718px; top: 30px; pointer-events: none; display: none;">
         <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 120 120"><defs><filter id="a" width="255.6%" height="255.6%" x="-77.8%" y="-77.8%" filterUnits="objectBoundingBox"><feGaussianBlur in="SourceGraphic" stdDeviation="14"></feGaussianBlur></filter></defs><circle cx="60" cy="60" r="27" fill="#7bbfd4" filter="url(#a)"></circle></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 695px; top: 35px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="8" height="8" viewBox="0 0 142 142">
              <circle cx="71" cy="71" r="71" fill="#fff2a8"></circle>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 680px; top: 36px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 201 201"><path d="M100.5 0.5L105.4 90.1 127.6 72.4 109.2 95.6 200.5 100.5 110 105.4 127.6 127.6 105.4 110 100.5 200.5 95.6 109.2 72.4 127.6 90.1 105.4 0.5 100.5 90.8 95.6 72.4 72.4 95.6 90.8 100.5 0.5Z" fill="#fff"></path></svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 637px; top: 29px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div><div class="jso-cursor-trail-shape" style="position: absolute; left: 566px; top: 3px; pointer-events: none; display: none;">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 336 336">
              <path fill="#ffcf5f" d="M168 243.6C128.4 243.6 88.8 258 49.2 286.8 78 247.2 92.4 207.6 92.4 168 92.4 128.4 78 88.8 49.2 49.2 88.8 78 128.4 92.4 168 92.4 207.6 92.4 247.2 78 286.8 49.2 258 88.8 243.6 128.4 243.6 168 243.6 207.6 258 247.2 286.8 286.8 247.2 258 207.6 243.6 168 243.6Z" transform="rotate(45 168 168)"></path>
            </svg>
        </div></div><template id="auto-clicker-autofill-popup">
  <style>
    @import url(https://cdn.jsdelivr.net/npm/bootstrap@5.3.0-alpha1/dist/css/bootstrap.min.css);
    @import url(https://fonts.googleapis.com/css?family=Poppins:300,400,500&subset=latin-ext);
    #body {
      background: white;
      border-radius: 5px;
      box-shadow: 0 30px 50px 0 rgb(44 49 59 / 20%);
      overflow: hidden;
      font-size: 14px;
      font-family: 'Poppins', sans-serif;
      min-width: 250px;
      max-width: 400px;
    }
    .modal-dialog {
      margin: auto;
    }
    .modal-content {
      border: none;
    }
    /*Header*/
    .modal-header {
      --bd-violet-rgb: 112.520718, 44.062154, 249.437846;
      padding-top: 0.5em;
      padding-bottom: 0.5em;
      padding-left: 0;
      height: 47px;
      background: rgba(112.520718, 44.062154, 249.437846, 0.1);
      color: #fff;
      border: none;
    }
    .modal-header button.btn {
      color: #712cf9;
      width: 1em;
      height: 1em;
      display: flex;
      box-sizing: content-box;
      justify-content: center;
      align-items: center;
      padding: 0;
      --bs-btn-close-opacity: 1;
      --bs-btn-close-hover-opacity: 0.75;
      --bs-btn-close-focus-shadow: 0 0 0 0.25em rgba(13, 110, 253, 0.25);
      opacity: var(--bs-btn-close-opacity);
    }
    .modal-header button.btn svg {
      width: inherit;
      height: inherit;
    }
    .modal-header button.btn:focus {
      box-shadow: var(--bs-btn-close-focus-shadow);
    }
    .modal-header button.btn:hover {
      opacity: var(--bs-btn-close-hover-opacity);
    }
    .modal-header button.expand .bi-arrows-expand {
      display: block !important;
    }
    .modal-header button.expand .bi-dash-lg {
      display: none;
    }
    .modal-header .modal-title {
      padding-left: 2em;
      color: #712cf9;
      position: relative;
      font-size: 1em;
    }
    .modal-header .modal-title::before {
      content: '';
      position: absolute;
      left: 0.75em;
      top: calc(50% - 0.375em);
      display: block;
      width: 0.75em;
      height: 0.75em;
      border-radius: 50%;
      background: red;
      cursor: pointer;
      box-shadow: 0 0 0 rgba(255, 0, 0, 0.4);
      animation: pulse 2s infinite;
    }
    @-webkit-keyframes pulse {
      0% {
        -webkit-box-shadow: 0 0 0 0 rgba(255, 0, 0, 0.4);
      }
      70% {
        -webkit-box-shadow: 0 0 0 10px rgba(255, 0, 0, 0);
      }
      100% {
        -webkit-box-shadow: 0 0 0 0 rgba(255, 0, 0, 0);
      }
    }
    @keyframes pulse {
      0% {
        -moz-box-shadow: 0 0 0 0 rgba(255, 0, 0, 0.4);
        box-shadow: 0 0 0 0 rgba(255, 0, 0, 0.4);
      }
      70% {
        -moz-box-shadow: 0 0 0 10px rgba(255, 0, 0, 0);
        box-shadow: 0 0 0 10px rgba(255, 0, 0, 0);
      }
      100% {
        -moz-box-shadow: 0 0 0 0 rgba(255, 0, 0, 0);
        box-shadow: 0 0 0 0 rgba(255, 0, 0, 0);
      }
    }
    /*Body*/
    .modal-body {
      max-height: calc(100vh - 7em);
      overflow-y: auto;
      overflow-x: hidden;
      padding: 0.5em 0;
    }
    td div {
      vertical-align: middle;
      width: 180px;
      line-height: 24px;
    }
    tr button {
      visibility: hidden;
    }
    tr:hover button {
      visibility: visible;
    }
    /*Footer*/
    .modal-footer {
      padding: 0;
      --bs-bg-opacity: 1;
      background-color: rgba(var(--bs-tertiary-bg-rgb), var(--bs-bg-opacity)) !important;
    }
    .modal-footer a {
      display: flex;
      align-items: center;
    }
    .modal-footer a svg {
      margin-right: 0.15em;
    }
    .modal-footer a:hover {
      color: #712cf9 !important;
    }
  </style>
  <div id="body" class="modal position-static d-block" data-bs-theme="light">
    <div class="modal-dialog">
      <div class="modal-content">
        <header class="modal-header">
          <h6 class="modal-title text-truncate">Auto Clicker - AutoFill</h6>
          <div class="d-flex justify-content-center align-items-center">
            <button type="button" class="btn ms-2" aria-label="collapse">
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-arrows-expand" style="display: none" viewBox="0 0 16 16">
                <path fill-rule="evenodd" d="M1 8a.5.5 0 0 1 .5-.5h13a.5.5 0 0 1 0 1h-13A.5.5 0 0 1 1 8ZM7.646.146a.5.5 0 0 1 .708 0l2 2a.5.5 0 0 1-.708.708L8.5 1.707V5.5a.5.5 0 0 1-1 0V1.707L6.354 2.854a.5.5 0 1 1-.708-.708l2-2ZM8 10a.5.5 0 0 1 .5.5v3.793l1.146-1.147a.5.5 0 0 1 .708.708l-2 2a.5.5 0 0 1-.708 0l-2-2a.5.5 0 0 1 .708-.708L7.5 14.293V10.5A.5.5 0 0 1 8 10Z"></path>
              </svg>
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-dash-lg" viewBox="0 0 16 16">
                <path fill-rule="evenodd" d="M2 8a.5.5 0 0 1 .5-.5h11a.5.5 0 0 1 0 1h-11A.5.5 0 0 1 2 8Z"></path>
              </svg>
            </button>
            <button type="button" class="btn ms-2" data-bs-dismiss="modal" aria-label="Close">
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-x-lg" viewBox="0 0 16 16">
                <path d="M2.146 2.854a.5.5 0 1 1 .708-.708L8 7.293l5.146-5.147a.5.5 0 0 1 .708.708L8.707 8l5.147 5.146a.5.5 0 0 1-.708.708L8 8.707l-5.146 5.147a.5.5 0 0 1-.708-.708L7.293 8 2.146 2.854Z"></path>
              </svg>
            </button>
          </div>
        </header>
        <main class="modal-body" id="collapse-main">
          <slot name="actions" hidden="">
            <table class="table table-sm table-hover mb-0">
              <thead>
                <tr>
                  <th scope="col">Name/Element</th>
                  <th scope="col">Value</th>
                  <th scope="col"></th>
                </tr>
              </thead>
              <tbody class="table-group-divider"></tbody>
            </table>
          </slot>
          <slot name="no-actions">
            <div class="px-2">
              <div class="card w-100">
                <div class="card-body">
                  <h5 class="card-title mb-3 text-primary">Start filling form...</h5>
                  <hr>
                  <h6 class="card-subtitle mb-2 text-muted">If you have already filled form</h6>
                  <button type="button" auto-generate-config="" class="btn btn-sm btn-outline-secondary">Already Filled ?</button>
                </div>
              </div>
            </div>
          </slot>
        </main>
        <footer class="modal-footer justify-content-center" id="collapse-footer">
          <ul class="nav justify-content-between w-100">
            <li class="nav-item">
              <a href="https://getautoclicker.com/docs/3.x/getting-started" class="nav-link px-2 text-muted" target="_blank">
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-file-richtext" viewBox="0 0 16 16">
                  <path d="M7 4.25a.75.75 0 1 1-1.5 0 .75.75 0 0 1 1.5 0zm-.861 1.542 1.33.886 1.854-1.855a.25.25 0 0 1 .289-.047l1.888.974V7.5a.5.5 0 0 1-.5.5H5a.5.5 0 0 1-.5-.5V7s1.54-1.274 1.639-1.208zM5 9a.5.5 0 0 0 0 1h6a.5.5 0 0 0 0-1H5zm0 2a.5.5 0 0 0 0 1h3a.5.5 0 0 0 0-1H5z"></path>
                  <path d="M2 2a2 2 0 0 1 2-2h8a2 2 0 0 1 2 2v12a2 2 0 0 1-2 2H4a2 2 0 0 1-2-2V2zm10-1H4a1 1 0 0 0-1 1v12a1 1 0 0 0 1 1h8a1 1 0 0 0 1-1V2a1 1 0 0 0-1-1z"></path>
                </svg>
                Docs
              </a>
            </li>
            <li class="nav-item">
              <a href="https://discord.gg/ubMBeX3" class="nav-link px-2 text-muted" target="_blank">
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-chat-fill" viewBox="0 0 16 16">
                  <path d="M8 15c4.418 0 8-3.134 8-7s-3.582-7-8-7-8 3.134-8 7c0 1.76.743 3.37 1.97 4.6-.097 1.016-.417 2.13-.771 2.966-.079.186.074.394.273.362 2.256-.37 3.597-.938 4.18-1.234A9.06 9.06 0 0 0 8 15z"></path>
                </svg>
                Chat
              </a>
            </li>
            <li class="nav-item">
              <a href="https://dev.getautoclicker.com/" class="nav-link px-2 text-muted" target="_blank" id="settings">
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-gear-fill" viewBox="0 0 16 16">
                  <path d="M9.405 1.05c-.413-1.4-2.397-1.4-2.81 0l-.1.34a1.464 1.464 0 0 1-2.105.872l-.31-.17c-1.283-.698-2.686.705-1.987 1.987l.169.311c.446.82.023 1.841-.872 2.105l-.34.1c-1.4.413-1.4 2.397 0 2.81l.34.1a1.464 1.464 0 0 1 .872 2.105l-.17.31c-.698 1.283.705 2.686 1.987 1.987l.311-.169a1.464 1.464 0 0 1 2.105.872l.1.34c.413 1.4 2.397 1.4 2.81 0l.1-.34a1.464 1.464 0 0 1 2.105-.872l.31.17c1.283.698 2.686-.705 1.987-1.987l-.169-.311a1.464 1.464 0 0 1 .872-2.105l.34-.1c1.4-.413 1.4-2.397 0-2.81l-.34-.1a1.464 1.464 0 0 1-.872-2.105l.17-.31c.698-1.283-.705-2.686-1.987-1.987l-.311.169a1.464 1.464 0 0 1-2.105-.872l-.1-.34zM8 10.93a2.929 2.929 0 1 1 0-5.86 2.929 2.929 0 0 1 0 5.858z"></path>
                </svg>
                Advance
              </a>
            </li>
            <li class="nav-item">
              <a href="https://github.com/sponsors/Dhruv-Techapps" class="nav-link px-2 text-muted" target="_blank">
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-balloon-heart text-danger" viewBox="0 0 16 16">
                  <path fill-rule="evenodd" d="m8 2.42-.717-.737c-1.13-1.161-3.243-.777-4.01.72-.35.685-.451 1.707.236 3.062C4.16 6.753 5.52 8.32 8 10.042c2.479-1.723 3.839-3.29 4.491-4.577.687-1.355.587-2.377.236-3.061-.767-1.498-2.88-1.882-4.01-.721L8 2.42Zm-.49 8.5c-10.78-7.44-3-13.155.359-10.063.045.041.089.084.132.129.043-.045.087-.088.132-.129 3.36-3.092 11.137 2.624.357 10.063l.235.468a.25.25 0 1 1-.448.224l-.008-.017c.008.11.02.202.037.29.054.27.161.488.419 1.003.288.578.235 1.15.076 1.629-.157.469-.422.867-.588 1.115l-.004.007a.25.25 0 1 1-.416-.278c.168-.252.4-.6.533-1.003.133-.396.163-.824-.049-1.246l-.013-.028c-.24-.48-.38-.758-.448-1.102a3.177 3.177 0 0 1-.052-.45l-.04.08a.25.25 0 1 1-.447-.224l.235-.468ZM6.013 2.06c-.649-.18-1.483.083-1.85.798-.131.258-.245.689-.08 1.335.063.244.414.198.487-.043.21-.697.627-1.447 1.359-1.692.217-.073.304-.337.084-.398Z"></path>
                </svg>
                Sponsors
              </a>
            </li>
          </ul>
        </footer>
