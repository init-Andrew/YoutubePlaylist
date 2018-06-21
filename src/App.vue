<template>
	<div id="app">
		<div class="nav-container">
			<nav>
				<img src="./assets/logo.png">
				<h1>Youtube Playlist</h1>
			</nav>
		</div>
		<div class="playlist-container">
			<div id="loader">
				<svg class="circular" viewBox="25 25 50 50">
					<circle class="path" cx="50" cy="50" r="20" fill="none" stroke-width="2" stroke-miterlimit="10"/>
				</svg>
			</div>
			<div class="video-thumbnail" @click="ytPlay">
				<div v-bind:style="{ 'background-image': 'url(https://img.youtube.com/vi/' + thumbnail + '/maxresdefault.jpg)' }"></div>
			</div>
			<div class="iframe-container">
				<div id="player"></div>
			</div>
			<div class="controls">
				<div class="control-block">
					<div class="progress">
						<div class="bar" v-bind:style="{ 'width': timePlayed + '%' }"></div>
					</div>
				</div>
				<div class="control-block">
					<ul>
						<li><a href="#" @click="ytBackward"><i class="fa fa-step-backward"></i></a></li>

						<li id="yt_play" v-show="!isPlaying"><a href="#" @click="ytPlay"><i class="fa fa-play"></i></a></li>
						<li id="yt_pause" v-show="isPlaying"><a href="#" @click="ytPause"><i class="fa fa-pause"></i></a></li>

						<li><a href="#" @click="ytForward"><i class="fa fa-step-forward"></i></a></li>
					</ul>
				</div>
				<div class="control-block">
					<label class="volume-control">
						<input type="range" min="0" max="100" step="1" @input="setVolume" @change="setVolume"/>
					</label>
				</div>
			</div>
			<div class="playlist">
				<ul>
					<li v-for="object in objectList" v-bind:class="{ active: isActive == object.url }">
						<a href="#" @click="ytLink" v-bind:data-url="object.url">{{ object.title }} <span>--:--</span></a>
					</li>
				</ul>
			</div>
		</div>
	</div>
</template>

<script>
export default {
    data() {
        return {
            objectList: [],
            defaultUrl: "",
            isActive: "",
            isPlaying: false,
            videoId: "",
            thumbnail: "",
            player: "",
            volume: 0,
            duration: 0,
            timePlayed: 0
        };
    },
    created() {
        this.$http
            .get("./links.json")
            .then(function(data) {
                return data.json();
            })
            .then(function(data) {
                //populate playlist and first videoID with data
                this.objectList = data.objectList;
                this.defaultUrl = data.objectList[0].url;

                //init yt api
                var tag = document.createElement("script");
                tag.src = "https://www.youtube.com/iframe_api";
                var firstScriptTag = document.getElementsByTagName("script")[0];
                firstScriptTag.parentNode.insertBefore(tag, firstScriptTag);

                //yt api ready call
                var self = this;
                window.onYouTubeIframeAPIReady = function() {
                    console.log("init IframeReady");
                    self.initIframe();
                };
            });
    },
    methods: {
        getVideoId(link) {
            var urlId = link.match(
                /(?:https?:\/{2})?(?:w{3}\.)?youtu(?:be)?\.(?:com|be)(?:\/watch\?v=|\/)([^\s&]+)/
            );
            return urlId[1];
        },
        initIframe() {
            var ytplayer;
            console.log("initframe URL " + this.defaultUrl);
            ytplayer = new YT.Player("player", {
                height: "390",
                width: "640",
                videoId: this.getVideoId(this.defaultUrl),
                playerVars: {
                    //'controls': 0,
                    //'enablejsapi': 1,
                    //'showinfo': 0,
                    rel: 0
                },
                events: {
                    onReady: this.playerReady,
                    onStateChange: this.playerState
                }
            });
            this.player = ytplayer;
        },
        playerReady() {
            setTimeout(() => {
                var loader = document.getElementById("loader");
                loader.classList.add("hidden");
            }, 150);
            this.showThumbnail();
            this.isActive = this.player.getVideoUrl();
            var controls = document.querySelector(".controls");
            controls.style.visibility = "visible";
        },
        playerState(e) {
            var state = e.data;
            var el = document.querySelector(".video-thumbnail");
            this.duration = this.player.getDuration();
            var duration = Number(Math.floor(this.duration) / 60.0).toFixed(2);
            var done = false;

            if (state == 1) {
                this.isPlaying = true;
                el.style.visibility = "hidden";
                if (duration != "undefined") {
                    var activeLink = document
                        .getElementById("app")
                        .querySelector(".active");
                    activeLink.childNodes[0].firstElementChild.innerText = duration.replace(
                        ".",
                        ":"
                    );
                    clearInterval(t);
                    var self = this;
                    let t = setInterval(function() {
                        self.showTime();
                    }, 1000);
                }
            } else {
                this.isPlaying = false;
            }
        },
        ytPlay(e) {
            e.preventDefault();
            this.player.playVideo();
            var activeLink =
                "https://www.youtube.com/watch?v=" +
                this.player.getVideoData()["video_id"];
            this.isActive = activeLink;
        },
        ytPause(e) {
            e.preventDefault();
            this.player.pauseVideo();
        },
        ytLink(e) {
            e.preventDefault();
            // load new video based on @click
            var linkId = e.target.dataset.url;
            this.player.loadVideoById({ videoId: this.getVideoId(linkId) });

            // set active link
            this.isActive = linkId;
        },
        ytForward(e) {
            e.preventDefault();
            this.timePlayed = 0;
            var lastEl = document.querySelector(".active")
                .parentElement.lastChild.children[0].dataset.url;
            var firstEl = document.querySelector(".active")
                .parentElement.firstChild.children[0].dataset.url;
            var active = document.querySelector(".active")
                .children[0].dataset.url;
            if (lastEl == active){
                this.player.loadVideoById({
                    videoId: this.getVideoId(firstEl)
                });
                this.isActive = firstEl;
                return;
            }
            var linkIdFoward = document.querySelector(".active")
                .nextElementSibling.children[0].dataset.url;


            this.player.loadVideoById({
                videoId: this.getVideoId(linkIdFoward)
            });
            // set active link
            this.isActive = linkIdFoward;
        },
        ytBackward(e) {
            e.preventDefault();
            // play new video based on previous element video id
            var firstEl = document.querySelector(".active")
                .parentElement.firstChild.children[0].dataset.url;
            var active = document.querySelector(".active")
                .children[0].dataset.url;
            var lastEl = document.querySelector(".active")
                .parentElement.lastChild.children[0].dataset.url;
            if(firstEl == active){
                this.player.loadVideoById({
                    videoId: this.getVideoId(lastEl)
                });
                this.isActive = lastEl;
                return;
            }

            var linkIdBackward = document.querySelector(".active")
                .previousElementSibling.children[0].dataset.url;

            this.player.loadVideoById({
                videoId: this.getVideoId(linkIdBackward)
            });

            // set active link
            this.isActive = linkIdBackward;
        },
        showThumbnail() {
            var el = document.querySelector(".video-thumbnail");
            el.style.visibility = "visible";
            var iframeId = this.player.getVideoData()["video_id"];
            this.thumbnail = iframeId;
        },
        setVolume(e) {
            this.volume = e.target.value;
            this.player.setVolume(this.volume);
        },
        showTime() {
            var duration = this.duration;
            var played = this.player.getCurrentTime();
            var time = Number(Math.floor(played / duration * 100).toFixed(2));
            if (typeof time === "number") {
                this.timePlayed = time;
            }
        }
    }
};
</script>

<style lang="scss">
* {
    box-sizing: border-box;
}
html {
    margin: 0;
    padding: 0;
    height: 100%;
}
body {
    padding: 0;
    margin: 0;
}
#app {
    font-family: "Raleway", sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #797a9b;
    .nav-container {
        text-align: center;
        width: 100%;
        height: 70px;
        background: #f1f1f1;
        border-bottom: 1px solid #dedede;
        display: block;
        text-align: left;
        nav {
            padding: 10px 0;
            max-width: 800px;
            margin: 0 auto;
            text-align: center;
            img {
                display: inline-block;
                height: 50px;
                float: left;
            }
            h1 {
                display: inline-block;
                font-size: 25px;
                font-weight: 400;
                margin: 0;
                margin-left: -85px;
                padding: 8px;
            }
        }
    }
    .playlist-container {
        background: #f1f1f1;
        border: 1px solid #dedede;
        max-width: 800px;
        margin: 40px auto;
        padding: 15px;
        position: relative;
        .video-thumbnail {
            visibility: hidden;
            position: absolute;
            left: 0;
            right: 0;
            top: 0;
            padding: 15px;
            div {
                background-size: cover;
                background-repeat: no-repeat;
                background-position: center center;
                display: block;
                height: 400px;
                cursor: pointer;
            }
        }
        .iframe-container {
            iframe {
                width: 100%;
                height: 400px;
            }
        }
        .controls {
            text-align: center;
            visibility: hidden;
            display: flex;
            max-width: 600px;
            margin: 0 auto;
            .control-block {
                display: inline-block;
                margin: 10px 0 0 0;
                flex-grow: 1;
                height: 100px;
                width: calc(100% * (1 / 3) - 10px - 1px);
                padding: 40px 0;
            }
            #yt_pause {
                a {
                    width: 40px;
                    height: 40px;
                    padding: 0;
                    border: 1px solid #797a9b;
                    border-radius: 100%;
                    i {
                        font-size: 21px;
                        text-align: center;
                        padding: 8px 0 0 0;
                    }
                }
            }
            #yt_play {
                a {
                    width: 40px;
                    height: 40px;
                    padding: 0;
                    border: 1px solid #797a9b;
                    border-radius: 100%;
                    display: inline-block;
                    i {
                        font-size: 24px;
                        text-align: center;
                        padding: 7px 0 0 5px;
                    }
                }
            }
            ul {
                list-style: none;
                display: block;
                margin-top: -9px;
                padding: 0;
                li {
                    display: inline-block;
                    vertical-align: middle;
                    width: 40px;
                    height: 40px;
                    margin: 0 5px;
                    a {
                        padding: 10px 0;
                        color: #797a9b;
                        cursor: pointer;
                        display: block;
                        i {
                            font-size: 17px;
                        }
                    }
                }
            }
            label.player-status {
                display: block;
                progress[value] {
                    -webkit-appearance: none;
                    -moz-appearance: none;
                    appearance: none;
                    /* Get rid of default border in Firefox. */
                    border: none;
                    width: 100%;
                    height: 2px;
                    background-color: #797a9b;
                    display: block;
                    margin-top: 10px;
                    &::-webkit-progress-bar {
                        border-radius: 2px;
                    }
                    &::-webkit-progress-value {
                        background-color: #f74f4f;
                        border-radius: 2px;
                    }
                    &::-moz-progress-bar {
                        background-color: #f74f4f;
                        border-radius: 2px;
                    }
                }
            }
            label.volume-control {
                display: block;
                input[type="range"] {
                    height: 10px;
                    border-radius: 3px;
                    background-color: transparent;
                    cursor: pointer;
                    border: none;
                    padding-left: 10px;
                    padding-right: 10px;
                    width: 100%;
                    position: relative;
                    -webkit-appearance: none;
                    -moz-appearance: none;
                    &::after {
                        content: "";
                        position: absolute;
                        top: 4px;
                        background-color: #797a9b;
                        height: 2px;
                        width: calc(100% - 10px * 2);
                        border-radius: 1px;
                        z-index: 1;
                        transition: background-color 100ms ease-in-out;
                    }
                    &::-moz-focus-outer {
                        border: 0;
                    }
                    &::-moz-range-track {
                        background-color: #797a9b;
                    }
                    &::-webkit-slider-thumb {
                        cursor: pointer;
                        width: 14px;
                        height: 14px;
                        border-radius: 50%;
                        border-color: transparent;
                        background-color: #797a9b;
                        outline: none !important;
                        position: relative;
                        z-index: 2;
                        transition: background-color 100ms ease-in-out;
                        -webkit-appearance: none;
                    }
                    &::-moz-range-thumb {
                        cursor: pointer;
                        width: 14px;
                        height: 14px;
                        border-radius: 50%;
                        border-color: transparent;
                        background-color: #797a9b;
                        outline: none !important;
                        position: relative;
                        z-index: 2;
                        transition: background-color 100ms ease-in-out;
                        -moz-appearance: none;
                    }
                    &::-ms-thumb {
                        cursor: pointer;
                        width: 14px;
                        height: 14px;
                        border-radius: 50%;
                        border-color: transparent;
                        background-color: #797a9b;
                        outline: none !important;
                        position: relative;
                        z-index: 2;
                        transition: background-color 60ms ease-in-out 15ms;
                    }
                    &:hover:after,
                    &:active:after {
                        background-color: #f74f4f;
                    }
                    &:hover::-moz-range-track,
                    &:active::-moz-range-track {
                        background-color: #f74f4f;
                    }
                    &:hover::-webkit-slider-thumb,
                    &:active::-webkit-slider-thumb {
                        background-color: #f74f4f;
                    }
                    &:hover::-moz-range-thumb,
                    &:active::-moz-range-thumb {
                        background-color: #f74f4f;
                    }
                    &:hover::-ms-thumb,
                    &:active::-ms-thumb {
                        background-color: #f74f4f;
                    }
                    &:active::-webkit-slider-thumb {
                        box-shadow: 0 0 0 2px #cc3300;
                    }
                    &:focus {
                        outline: none !important;
                    }
                }
            }
            .progress {
                overflow: hidden;
                width: 100%;
                height: 2px;
                background-color: #797a9b;
                display: block;
                margin-top: 10px;
                .bar {
                    width: 50%;
                    height: 18px;
                    background-color: #f74f4f;
                    -webkit-box-sizing: border-box;
                    -moz-box-sizing: border-box;
                    box-sizing: border-box;
                    transition: all 200ms ease-in-out;
                }
            }
        }
        .playlist {
            ul {
                list-style: none;
                display: block;
                margin: 0;
                padding: 30px 0;
                li {
                    padding: 10px 0;
                    margin: 0 -15px;
                    border-bottom: 1px solid #dedede;
                    &.active {
                        background: #e3e3e3;
                        a {
                            color: #3f4069;
                        }
                    }
                    a {
                        display: block;
                        padding: 0 10px 0 15px;
                        color: #797a9b;
                        text-decoration: none;
                        cursor: pointer;
                        overflow: hidden;
                        span {
                            font-family: Arial, Helvetica, sans-serif;
                            float: right;
                            font-weight: 400;
                        }
                    }
                }
            }
        }
    }
    #loader {
        background: #f1f1f1;
        position: absolute;
        left: 0;
        right: 0;
        top: 0;
        bottom: 0;
        z-index: 2;

        &:before {
            content: "";
            display: block;
            padding-top: 100%;
        }
        &.hidden {
            visibility: hidden;
            opacity: 0;
            transition: visibility 0s 250ms, opacity 250ms linear;
        }
    }

    .circular {
        animation: rotate 2s linear infinite;
        height: 60px;
        transform-origin: center center;
        width: 60px;
        position: absolute;
        top: 25%;
        left: 0;
        right: 0;
        margin: auto;
    }

    .path {
        stroke-dasharray: 1, 200;
        stroke-dashoffset: 0;
        animation: dash 1.5s ease-in-out infinite, color 6s ease-in-out infinite;
        stroke-linecap: round;
    }
    @keyframes rotate {
        100% {
            transform: rotate(360deg);
        }
    }

    @keyframes dash {
        0% {
            stroke-dasharray: 1, 200;
            stroke-dashoffset: 0;
        }
        50% {
            stroke-dasharray: 89, 200;
            stroke-dashoffset: -35px;
        }
        100% {
            stroke-dasharray: 89, 200;
            stroke-dashoffset: -124px;
        }
    }

    @keyframes color {
        100%,
        0% {
            stroke: #d62d20;
        }
        40% {
            stroke: #0057e7;
        }
        66% {
            stroke: #008744;
        }
        80%,
        90% {
            stroke: #ffa700;
        }
    }
}
</style>
