<script>
    import { onMount } from 'svelte';
    import KissFFT from 'kissfft-js';

    let video;
    let canvas;
    let canvasCtx;
    let hrIndicator;
    let graph;

    const boxsize = 4;
    let boxLeft;
    let boxTop;
    let boxWidth;
    let boxHeight;

    let heartrate = 0;
    let maxHistLen = 64;
    let bloodHist = Array(maxHistLen).fill(0);
    let timingHist = Array(maxHistLen).fill(0);
    let last = performance.now();
    let fft = new KissFFT.FFTR(maxHistLen);
    var average = (array) => array.reduce((a, b) => a + b) / array.length;

    async function calcHeartbeat() {
        let bloodRegion = canvasCtx.getImageData(boxLeft, boxTop, boxWidth, boxHeight);

        let videoDataSum = bloodRegion.data.reduce((a, b) => a + b, 0);
        videoDataSum -= boxWidth * boxHeight * 255;
        let avgIntensity = videoDataSum / (boxWidth * boxHeight * 3);

        timingHist.push(1 / ((performance.now() - last) * 0.001));
        last = performance.now();

        bloodHist.push(bloodHist[maxHistLen - 1]*0.8 + 0.2*avgIntensity);
        if (bloodHist.length > maxHistLen) {
            bloodHist.shift();
            timingHist.shift();

            let fftData = await calcFFT(bloodHist);
            update(timingHist, fftData);
            updateGraph(bloodHist);
        }
    }

    async function calcFFT(data) {
        const avg = average(data);
        data = data.map(elem => elem - avg);
        let tmp = fft.forward(data);
        return tmp.slice(1);
    }

    function update(times, data) {
        data = data.map(elem => Math.abs(elem));
        let curPollFreq = average(times.slice(Math.round(maxHistLen / 2)));
        let binNumber = Array.from(data).map((elem, index) => index + 1);
        let binHz = binNumber.map(elem => elem * curPollFreq / maxHistLen);

        let maxVal = 0
        let maxHz = 0;
        let maxInd = 0;

        for (let i = 0; i < binHz.length; i++) {
            if (binHz[i] > 0.66 && binHz[i] < 2) {
                if (data[i] > maxVal) {
                    maxVal = data[i];
                    maxHz = binHz[i];
                    maxInd = i;
                }
            }
        }

        heartrate += (maxHz - heartrate) * 0.03;
        hrIndicator.innerHTML = `<i class="fas fa-heartbeat"></i>&nbsp;&nbsp;${Math.round(heartrate * 60)}&nbsp;BPM`;
    }

    function updateGraph(data) {
        let indexedData = Array.from(data).map((elem, index) => [index + 1, elem])

        new Dygraph(graph, indexedData);
    }

    onMount(async() => {
        if (/Android|webOS|iPhone|iPad|iPod|like Mac OS X/i.test(navigator.userAgent)) {
            alert('모바일 환경에서는 정상적으로 작동하지 않습니다.\n웹캠이 장착된 PC나 노트북을 이용하세요.');
        }

        video = document.getElementById('input_video');
        canvas = document.getElementById('output_canvas');
        hrIndicator = document.getElementById('hr_indicator')
        graph = document.getElementById('graph');
        canvasCtx = canvas.getContext('2d');

        const faceMesh = new FaceMesh({
            locateFile: (file) => {
                return `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`;
            }
        });

        faceMesh.setOptions({
            maxNumFaces: 1,
            refineLandmarks: true,
            minDetectionConfidence: 0.5,
            minTrackingConfidence: 0.5
        });

        faceMesh.onResults(async(results) => {
            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvas.width, canvas.height);
            canvasCtx.drawImage(results.image, 0, 0, canvas.width, canvas.height);

            if (results.multiFaceLandmarks) {
                const landmarks = results.multiFaceLandmarks[0];

                if (landmarks === undefined) return;

                drawConnectors(canvasCtx, landmarks, FACEMESH_TESSELATION, {color: '#C0C0C070', lineWidth: 1});
				drawConnectors(canvasCtx, landmarks, FACEMESH_RIGHT_EYE, {color: '#30FF30', lineWidth: 1});
				drawConnectors(canvasCtx, landmarks, FACEMESH_RIGHT_EYEBROW, {color: '#30FF30', lineWidth: 1});
				drawConnectors(canvasCtx, landmarks, FACEMESH_RIGHT_IRIS, {color: '#30FF30', lineWidth: 1});
				drawConnectors(canvasCtx, landmarks, FACEMESH_LEFT_EYE, {color: '#30FF30', lineWidth: 1});
				drawConnectors(canvasCtx, landmarks, FACEMESH_LEFT_EYEBROW, {color: '#30FF30', lineWidth: 1});
				drawConnectors(canvasCtx, landmarks, FACEMESH_LEFT_IRIS, {color: '#30FF30', lineWidth: 1});
				drawConnectors(canvasCtx, landmarks, FACEMESH_FACE_OVAL, {color: '#E0E0E0'});
				drawConnectors(canvasCtx, landmarks, FACEMESH_LIPS, {color: '#E0E0E0', lineWidth: 1});

                boxLeft = landmarks[117].x;
                boxTop = landmarks[117].y;
                boxWidth = landmarks[346].x - boxLeft;
                boxHeight = landmarks[164].y - boxTop;

                boxLeft *= 640;
                boxTop *= 360;
                boxWidth *= 640;
                boxHeight *= 360;

                canvasCtx.strokeStyle = '#FF0000';
                canvasCtx.lineWidth = 3;
                canvasCtx.beginPath();
                canvasCtx.rect(boxLeft - boxsize, boxTop - boxsize, boxWidth + boxsize * 2, boxHeight + boxsize * 2);
                canvasCtx.stroke();
            }

            canvasCtx.restore();

            await calcHeartbeat();
        });

        const camera = new Camera(video, {
            onFrame: async() => {
                await faceMesh.send({
                    image: video
                });
            },
            width: 1280,
            height: 720
        });

        camera.start();
    });
</script>

<main>
    <div class="container">
        <br/>
        <br/>
        <h1 class="title">웹캠을 이용한 심박수 측정</h1>
        <div class="columns is-vcentered">
            <div class="column">
                <video id="input_video" width="640px" height="360px" style="display: none;"></video><br/>
                <canvas id="output_canvas" width="640px" height="360px"></canvas><br/>
            </div>
            <div class="column">
                <div id="graph"></div>
            </div>
        </div>
        <div class="content">
            <h2 class="subtitle" id="hr_indicator" style="color: hsl(348, 100%, 61%);"><i class="fas fa-heartbeat"></i>&nbsp;&nbsp;--&nbsp;BPM</h2>
            <ul>
                <li>화면에 표시되는 심박수는 추정치이며 실제 심박수와는 차이가 있습니다.</li>
                <li>웹캠과 얼굴이 고정되어 있어야 정확도가 높습니다.</li>
            </ul>
        </div>
        <nav class="level">
            <div class="level-left"></div>
            <div class="level-right">
                <a class="button level-item is-link" href="/paper.pdf"><i class="far fa-file-pdf"></i>&nbsp;소논문 (pdf)</a>
                <a class="button level-item is-link" href="/paper.hwp"><i class="far fa-file"></i>&nbsp;소논문 (hwp)</a>
                <a class="button level-item is-black" href="https://github.com/sangwon090/ppg/"><i class="fab fa-github"></i>&nbsp;소스코드 (Github)</a>
            </div>
        </nav>
    </div>
</main>