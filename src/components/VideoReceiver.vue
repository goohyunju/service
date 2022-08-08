<template>
    <div id="media">
        <div id="videos">
            <div id="video0">
                <video id="remote-video0" class="remote-video"></video>
            </div>
            <div id="video1">
                <video id="remote-video1" class="remote-video"></video>
            </div>
            <div id="video2">
                <video id="remote-video2" class="remote-video"></video>
            </div>
            <div id="video3">
                <video id="remote-video3" class="remote-video"></video>
            </div>
        </div>
    </div>
</template>


<script>
import Vue from 'vue'

Vue.prototype.eventBus = new Vue();

export default {
    name: 'VideoReceiver',
    props:{
    },
    data(){
        return{
            room: "",
            joined: false,
            hasPeer: false,
            isStarted: false,
            pc:null,
            remoteStreams: new Array(),
            remoteStreamIndex: 0,
            pcConfig: {
              iceServers: []
            },
        }
    },
    mounted(){
        this.room = this.$route.params.room || prompt("[User] Room을 입력해주세요.(ex. agent ID)", "");
        this.$vsocket.emit('hi', this.room, 'VR');
        this.$vsocket.emit('ice');
        
        // iceserver receiving
        this.$vsocket.on('ice', iceServers => {
          this.pcConfig.iceServers=iceServers;
          console.log('ice was loaded from signal server : ' + JSON.stringify(this.pcConfig.iceServers));
          this.createPeerConnection();
        });
        
        this.$vsocket.on('joined', room => { 
            console.log('joined ' + room);
            this.joined = true;
        });

        this.$vsocket.on('full', room => {
            console.log('Room ' + room + ' is full');
        });
        
        this.$vsocket.on('leaved', room => {
            console.log('leaved ' + room);
            this.joined = false;
        });

        this.$vsocket.on('peer joined', socketId => {
            console.log('peer joined : ' + socketId);
            if (!this.pcConfig.iceServers[0]) {
                console.log('ice server is not set yet');
                return
            }
            if (this.hasPeer) {
              this.sendMessage('ready monitoring video');
            }
            
        });
        
        this.$vsocket.on('peer leaved', socketId => {
            console.log('video sender leaved : ' + socketId);
            // this.hasPeer = false;
            
        });

        this.$vsocket.on('log', array => {
            console.log.apply(console, array);
        });

        // This client receives a message
        this.$vsocket.on('message', (message, socketId) => {
            console.log('Client received message:', message);
            
            if (message.type === 'offer' && this.hasPeer && !this.isStarted) {
                this.pc.setRemoteDescription(new RTCSessionDescription(message));
                this.doAnswer();
                this.isStarted = true;
                
            } else if (message.type === 'candidate' && this.hasPeer) {
                var candidate = new RTCIceCandidate({
                    sdpMLineIndex: message.label,
                    candidate: message.candidate
                });
                this.pc.addIceCandidate(candidate);
            } else if(message === 'ready remote video'){
                if (this.joined && this.hasPeer) {
                    if (this.isStarted) {
                         console.log('Im already started');
                    } else {
                        this.resetMedia();
                        this.sendMessage('ready monitoring video');
                        console.log('ready monitoring video');
                    }
                }
            } else if (message === 'bye remote video' && this.hasPeer) {
                this.handleRemoteHangup();
            }
        });
        this.eventBus.$on('reload-video',()=>{
            
            this.sendMessage('reload');

        });
        window.onbeforeunload = () => {
	  this.sendMessage("bye monitoring video");
	  this.$vsocket.emit('bye', this.room, 'VR');
	};
    },
    methods: {
        /*    common    */
        sendMessage(message) {
            console.log('Client sending message: ', message);
            this.$vsocket.emit('message', message);
        },
        createPeerConnection() {
            try {
                this.pc = new RTCPeerConnection(this.pcConfig);
                this.pc.onicecandidate = this.handleIceCandidate;
                this.pc.onaddstream = this.handleRemoteStreamAdded;
                this.pc.onremovestream = this.handleRemoteStreamRemoved;
                this.hasPeer = true;
                console.log('Created RTCPeerConnnection');
            } catch (e) {
                console.error('Failed to create PeerConnection, exception: ' + e.message);
                return;
            }
        },
        removePeerConnection() {
            try {
                this.pc.close();
                this.pc = null;
                this.hasPeer = false;
                console.log('Removed RTCPeerConnection');
            } catch (e) {
                console.error('Failed to remove PeerConnection, exception : ' + e.message);
                return
            }
        },
        setLocalAndSendMessage(sessionDescription) {
            this.pc.setLocalDescription(sessionDescription);
            console.log('setLocalAndSendMessage sending message', sessionDescription);
            this.sendMessage(sessionDescription);
        },
        handleIceCandidate(event) {
            // console.log('icecandidate event: ', event);
            if (event.candidate) {
                this.sendMessage({
                type: 'candidate',
                label: event.candidate.sdpMLineIndex,
                id: event.candidate.sdpMid,
                candidate: event.candidate.candidate
                });
            } else {
                console.log('End of candidates.');
            }
        },
        stop() {
            this.pc.close();
            this.pc = null;
            this.isStarted = false;
        },
        /* */
        doAnswer() {
            console.log('Sending answer to peer.');
            this.pc.createAnswer().then(
                this.setLocalAndSendMessage,
                this.onCreateSessionDescriptionError
            );
        },
        onCreateSessionDescriptionError(error) {
            //   trace('Failed to create session description: ' + error.toString());
            console.log('Failed to create session description: ',error.toString());
        },
        handleRemoteStreamAdded(event) {
            console.log('remote stream added : ' + JSON.stringify(event.stream))
            var mediaKind = event.stream.getTracks()[0].kind;
            
            if (mediaKind == 'video') {
                var video = document.querySelector('#remote-video' + this.remoteStreamIndex);
                if(video == null)
                    return;
                this.remoteStreams[this.remoteStreamIndex] = event.stream;
                video.srcObject = this.remoteStreams[this.remoteStreamIndex++];
                video.controls = true;
                video.autoplay = true;
            }
        },
        handleRemoteStreamRemoved(event) {
            console.log('Remote stream removed. Event: ', event);
            
        },
        handleRemoteHangup() {
            this.stop();
            //this.hasPeer = false;
        },
        resetMedia(){
            this.remoteStreams = [];
            this.remoteStreamIndex = 0;
            console.log("Reset Media");
            //this.pc = null;
            //this.hasPeer = false;
        }



    }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#media{
    display: block;
    flex: 1;
    font-size: 0;
}
#videos{
    display: block;
    height: 50vh;
}
  
#videos div{
    display: inline-block;
    position: relative;
    width: 50%;
    height: 50%;
}

.remote-video{
    position: absolute;
    width: 100%;
    height: 100%;
    object-fit: fill;
    outline: none;
}
</style>
