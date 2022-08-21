<template>
    <div>
        <div class="min-h-[calc(100vh-120px)]">
            <div id="camera" class="h-[250px]"></div>
            <div class="controllers">
                <select @change="changeVideoSource">
                    <option>
                        {{
                            ctrls.lang === "tr" ?
                            "Kamera seç" :
                            "Choose cam"
                        }}
                    </option>
                    <option v-for="device in videoDevices" :key="device" :value="device.deviceId">
                        {{ device.label.substring(0, 12) }}
                    </option>
                </select>
                <button @click="mirrorVideo">
                    {{
                        ctrls.lang === "tr" ?
                        "Aynala" :
                        "Mirror"
                    }}
                </button>
                <button @click="refresh">
                    {{
                        ctrls.lang === "tr" ?
                        "Yenile" :
                        "Refresh"
                    }}
                </button>
                <button @click="changeLang">{{ ctrls.lang.toUpperCase() }}</button>
            </div>
            <div id="meaning" class="min-h-[250px]">
                {{
                    ctrls.lang === "tr" ?
                    motif["info-tr"] || "Anlamını görmek için bir motifi tarayabilirsiniz!" :
                    motif["info-en"] || "You can scan a motif to see it's meaning!" 
                }}
            </div>
        </div>
        <footer>
            <div>
                {{
                    ctrls.lang === "tr" ?
                    "Teknofest Turizm Teknolojileri Yarışması 2022 Finalisti Argumareot Takımı" :
                    "Teknofest Tourism Technologies Competition 2022 Finalist Argumareot Team" 
                }}
            </div>
            <div>Made with 🖤 open source on <a href="https://github.com/orhanemree/argumareot-turizm">GitHub</a>.</div>
        </footer>
    </div>
</template>

<script>
import data from "./assets/data.json";
export default {
    data(){
        return{
            ctrls: { lang: "tr", sourceChanged: false, mirrored: false, scanned: false },
            videoSource: {},
            videoDevices: [],
            motif: {}
        }
    },
    methods: {
        getVideoDevices(){
            navigator.mediaDevices.enumerateDevices().then(devices => {
                    this.videoDevices = devices.filter(device => device.kind === "videoinput");
                }
            );
        },
        changeVideoSource(e){
            this.videoSource = { video: { deviceId: e.target.value } };
            this.ctrls.sourceChanged = true;
        },
        mirrorVideo(){
            this.ctrls.mirrored = !this.ctrls.mirrored;
            const canvas = document.querySelector("#camera canvas");
            canvas.classList.toggle("-scale-x-100");
        },
        refresh(){
            location.reload();
        },
        changeLang(){
            this.ctrls.lang = this.ctrls.lang === "tr" ? "en" : "tr";
        }
    },
    mounted(){
        this.getVideoDevices();
        if ("serviceWorker" in navigator) {
            navigator.serviceWorker.register("/service-worker.js");
        }
        const MODEL_URL = "https://teachablemachine.withgoogle.com/models/EKbsPs3Cl/model.json";
        let classifier, videoInput, outputWidth, outputHeight;
        // video classify with ml5.js
        const classifyVideo = () => {
            classifier.classify(videoInput, (err, result) => {
                if (!err && result[0].confidence * 100 > 95 && result[0].label !== "light" && result[0].label !== "dark"){
                    this.motif = data.filter(m => m.name === result[0].label)[0];
                    this.ctrls.scanned = true;
                }
                if (!this.ctrls.scanned){
                    classifyVideo();
                }
            });
        }
        // p5.js functions
        const script = p => {
            p.preload = _ => {
                classifier = ml5.imageClassifier(MODEL_URL);
            }
            p.setup = _ => {
                outputWidth = window.innerWidth - 100;
                outputHeight = 250;
                const canvas = p.createCanvas(outputWidth, outputHeight);
                canvas.parent("camera");
                videoInput = p.createCapture(p.VIDEO);
                videoInput.size(outputWidth, outputHeight);
                videoInput.hide();
                classifyVideo();
            }
            
            p.draw = _ => {
                if (this.ctrls.sourceChanged){
                    videoInput = p.createCapture(this.videoSource || p.VIDEO);
                    videoInput.size(outputWidth, outputHeight);
                    videoInput.hide();
                    this.ctrls.sourceChanged = false;
                }
                p.image(videoInput, 0, 0, outputWidth, outputHeight);
            }
        }
        new p5(script);
    }
}
</script>