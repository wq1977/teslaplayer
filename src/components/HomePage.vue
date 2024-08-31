<script setup lang="ts">
import { ref, onMounted, nextTick } from 'vue'
const videos = ref([])
const songs = ref([])
const videoPlayer = ref(null)
const currentSong = ref({})
const isPlaying = ref(false)
const isLoading = ref(false)
const showPhoneScreen = ref(false)
const videoProgress = ref(0)
const mirrorUrl = ref('')

const prefix = import.meta.env.MODE === 'development' ? 'http://localhost:3000' : ''

const showToast = ref(false)
const toastMessage = ref('')

function displayToast(message, duration = 3000) {
    toastMessage.value = message
    showToast.value = true
    setTimeout(() => {
        showToast.value = false
    }, duration)
}


onMounted(() => {
    fetch(`${prefix}/list`)
        .then(res => res.json())
        .then(data => {
            data = data.files.map(song => ({
                ...song,
                url: `${prefix}/media/${song.path}`,
                name: song.name || song.path.split('/').pop().replace(/\.[^/.]+$/, '').replace('_karaoke', '')
            })).filter(song => song.path.toLowerCase().endsWith('.mp4') || song.path.toLowerCase().endsWith('.mkv'))
            songs.value = data.filter(song => song.path.indexOf('kalaok') > -1 || song.path.indexOf('_karaoke') > -1)
            videos.value = data.filter(song => !(songs.value.filter(s => s.path == song.path)[0]))
        })
    mirrorUrl.value = `http://${location.host.split(':')[0]}:3339`
})

function playSong(song) {
    currentSong.value = song
    isPlaying.value = true
    videoPlayer.value.src = song.url
    // 确保视频元素已经更新
    nextTick(() => {
        if (videoPlayer.value) {
            videoPlayer.value.load() // 重新加载视频源
            videoPlayer.value.play() // 开始播放新的视频
        }
    })
}

function togglePlay() {
    if (isPlaying.value) {
        videoPlayer.value.pause()
    } else {
        videoPlayer.value.play()
    }
}

function playKalaOK() {
    currentSong.value = songs.value.sort(() => Math.random() - 0.5)
    isPlaying.value = true
    nextTick(() => {
        if (videoPlayer.value) {
            if (videoPlayer.value.src && videoPlayer.value.src != currentSong.value[0].url) {
                videoPlayer.value.src = currentSong.value[0].url
            }
            videoPlayer.value.load()
            videoPlayer.value.play()
        }
    })
}

function playPrevious() {
    const sources = videoPlayer.value.getElementsByTagName('source');
    if (sources.length > 1) {
        const currentIndex = Array.from(sources).findIndex(source => source.src === videoPlayer.value.currentSrc);
        const previousIndex = (currentIndex - 1 + sources.length) % sources.length;
        videoPlayer.value.src = sources[previousIndex].src;
        videoPlayer.value.load();
        videoPlayer.value.play();
    }
}

function playNext() {
    const sources = videoPlayer.value.getElementsByTagName('source');
    if (sources.length > 1) {
        const currentIndex = Array.from(sources).findIndex(source => source.src === videoPlayer.value.currentSrc);
        const nextIndex = (currentIndex + 1) % sources.length;
        videoPlayer.value.src = sources[nextIndex].src;
        videoPlayer.value.load();
        videoPlayer.value.play();
    }
}

function updateProgress() {
    videoProgress.value = videoPlayer.value.currentTime / videoPlayer.value.duration * 100
}

function updateVideoTime() {
    videoPlayer.value.currentTime = videoPlayer.value.duration * videoProgress.value / 100;
}

</script>
<template>
    <div class="video-container" style="position: relative; width: 100vw; height: 100vh;">
        <video ref="videoPlayer" @click="togglePlay" class="full-screen-video" @play="isPlaying = true"
            @pause="isPlaying = false" @waiting="isLoading = true" @canplay="isLoading = false" @ended="playNext"
            @timeupdate="updateProgress"
            style="position: absolute; background-color: black; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 100%; height: 100%; object-fit: contain; z-index: 1;">
            <source v-if="currentSong.url" :src="currentSong.url" type="video/mp4">
            <source v-for="song in currentSong.length ? currentSong : []" :src="song.url" type="video/mp4">
        </video>
        <div v-if="isLoading" class="loading-overlay"
            style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 2; display: flex; justify-content: center; align-items: center;">
            <div class="spinner"
                style="width: 50px; height: 50px; border: 5px solid #f3f3f3; border-top: 5px solid #3498db; border-radius: 50%; animation: spin 1s linear infinite;">
            </div>
        </div>


        <div v-if="!isPlaying" class="phone-button" @click="showPhoneScreen = !showPhoneScreen"
            :class="{ 'active': showPhoneScreen }">
            <div class="phone-icon">
                <div class="phone-screen"></div>
                <div class="phone-home-button"></div>
            </div>
        </div>

        <div v-if="showPhoneScreen" class="phone-screen-overlay">
            <!-- 这里可以添加手机屏幕的内容 -->
            <div class="phone-screen-content">
                <iframe :src="mirrorUrl" class="phone-iframe" style="
                width: 100%;
                height: 100%;
                border: none;
                margin: 0;
                padding: 0;
                border-radius: 0;
                box-shadow: 0 0 20px rgba(0, 0, 0, 0.2);
                background-color: #f5f5f5;
                transition: all 0.3s ease-in-out;
            "></iframe>
            </div>
        </div>

        <div class="playback-controls" v-if="currentSong.url && !isPlaying"
            style="position: absolute; bottom: 10%; left: 50%; transform: translateX(-50%); z-index: 3; display: flex; align-items: center; background-color: rgba(255, 255, 255, 0.2); padding: 10px; border-radius: 20px; backdrop-filter: blur(5px);">
            <button @click="togglePlay" class="fancy-button"
                style="background: linear-gradient(45deg, #ff6b6b, #feca57); border: none; color: white; width: 40px; height: 40px; border-radius: 50%; margin-right: 15px; display: flex; justify-content: center; align-items: center; box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); transition: all 0.3s ease;">
                <span :style="{ fontSize: '18px' }">
                    {{ isPlaying ? '⏸' : '▶' }}
                </span>
            </button>
            <input type="range" v-model="videoProgress" @input="updateVideoTime" min="0" max="100"
                class="fancy-progress"
                style="width: 70vw; appearance: none; -webkit-appearance: none; background: linear-gradient(to right, #4a69bd, #0abde3); height: 8px; border-radius: 4px; outline: none; opacity: 0.7; transition: opacity 0.2s;">
        </div>

        <div v-if="showToast" class="toast-container">
            <div class="toast-message">
                {{ toastMessage }}
            </div>
        </div>

        <div class="video-controls" v-if="currentSong.length && !isPlaying"
            style="position: absolute; bottom: 10%; left: 50%; transform: translateX(-50%); z-index: 3; display: flex; justify-content: center; align-items: center;">
            <button @click="playPrevious" class="control-button"
                style="background-color: rgba(255, 255, 255, 0.2); border: none; color: white; padding: 10px 20px; margin: 0 10px; border-radius: 25px; font-size: 16px; backdrop-filter: blur(5px); transition: all 0.3s ease;">
                <i class="fas fa-step-backward"> </i> 上一首
            </button>
            <button @click="togglePlay" class="control-button"
                style="background-color: rgba(255, 255, 255, 0.2); border: none; color: white; padding: 10px 20px; margin: 0 10px; border-radius: 25px; font-size: 16px; backdrop-filter: blur(5px); transition: all 0.3s ease;">
                <i :class="isPlaying ? 'fas fa-pause' : 'fas fa-play'"> </i> {{ isPlaying ? '暂停' : '播放' }}
            </button>
            <button @click="playNext" class="control-button"
                style="background-color: rgba(255, 255, 255, 0.2); border: none; color: white; padding: 10px 20px; margin: 0 10px; border-radius: 25px; font-size: 16px; backdrop-filter: blur(5px); transition: all 0.3s ease;">
                下一首 <i class="fas fa-step-forward"></i>
            </button>
        </div>

        <div v-if="!isPlaying" class="song-list-overlay"
            style="position: absolute; padding: 50px; top: 0; left: 0; right: 0; bottom:0; z-index: 2; overflow-y: auto;">
            <div @click="playKalaOK()" v-if="songs.length" class="song-item" :style="{
                display: 'flex',
                justifyContent: 'space-between',
                alignItems: 'center',
                padding: '10px 15px',
                margin: '10px 0',
                backgroundColor: 'rgba(255, 255, 255, 0.1)',
                borderRadius: '8px',
                boxShadow: '0 4px 6px rgba(0, 0, 0, 0.1)',
                transition: 'all 0.3s ease',
                cursor: 'pointer'
            }" @mouseover="$event.target.style.transform = 'scale(1.05)'"
                @mouseout="$event.target.style.transform = 'scale(1)'">
                <span class="song-name" :style="{
                    fontSize: '18px',
                    fontWeight: 'bold',
                    color: '#fff'
                }">开始 K 歌</span>
                <span class="song-icon" :style="{
                    fontSize: '24px',
                    opacity: '0.8'
                }">🎤</span>
            </div>
            <div v-for="song in videos" :key="song.path" @click="playSong(song)" class="song-item" :style="{
                display: 'flex',
                justifyContent: 'space-between',
                alignItems: 'center',
                padding: '10px 15px',
                margin: '10px 0',
                backgroundColor: 'rgba(255, 255, 255, 0.1)',
                borderRadius: '8px',
                boxShadow: '0 4px 6px rgba(0, 0, 0, 0.1)',
                transition: 'all 0.3s ease',
                cursor: 'pointer'
            }" @mouseover="$event.target.style.transform = 'scale(1.05)'"
                @mouseout="$event.target.style.transform = 'scale(1)'">
                <span class="song-name" :style="{
                    fontSize: '18px',
                    fontWeight: 'bold',
                    color: '#fff'
                }">{{ song.name }}</span>
                <span class="song-icon" :style="{
                    fontSize: '24px',
                    opacity: '0.8'
                }">🎥</span>
            </div>
        </div>
    </div>
</template>

<style>
@keyframes spin {
    0% {
        transform: rotate(0deg);
    }

    100% {
        transform: rotate(360deg);
    }
}

.toast-container {
    position: fixed;
    top: 20px;
    left: 50%;
    transform: translateX(-50%);
    z-index: 9999;
}

.toast-message {
    background-color: rgba(0, 0, 0, 0.7);
    color: white;
    padding: 12px 20px;
    border-radius: 25px;
    font-size: 16px;
    backdrop-filter: blur(5px);
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    animation: fadeIn 0.3s, fadeOut 0.3s 2.7s;
}

@keyframes fadeIn {
    from {
        opacity: 0;
        transform: translateY(-20px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes fadeOut {
    from {
        opacity: 1;
        transform: translateY(0);
    }

    to {
        opacity: 0;
        transform: translateY(-20px);
    }
}

.phone-button {
    position: fixed;
    bottom: 20px;
    right: 20px;
    width: 60px;
    height: 100px;
    background-color: #333;
    border-radius: 10px;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    transition: all 0.3s ease;
    z-index: 1000;
}

.phone-button:hover {
    transform: scale(1.1);
    box-shadow: 0 0 15px rgba(255, 255, 255, 0.5);
}

.phone-button.active {
    background-color: #4a69bd;
}

.phone-icon {
    width: 50px;
    height: 90px;
    background-color: #222;
    border-radius: 8px;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    align-items: center;
    padding: 5px;
}

.phone-screen {
    width: 45px;
    height: 75px;
    background-color: #4a69bd;
    border-radius: 3px;
}

.phone-home-button {
    width: 15px;
    height: 15px;
    border: 2px solid #666;
    border-radius: 50%;
}

.phone-screen-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.8);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 999;
}

.phone-screen-content {
    background-color: white;
    padding: 0;
    margin: 0;
    border-radius: 0;
    text-align: center;
    width: 100%;
    height: 100%;
}
</style>