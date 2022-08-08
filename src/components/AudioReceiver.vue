<template>
    <div id="media">
        <div id="remote-audios">
        </div>
    </div>
</template>


<script>
import Vue from 'vue'

Vue.prototype.eventBus = new Vue();

export default {
    name: 'AudioReceiver',
    props:{
    },
    data(){
        return{
            room: "",
            joined: false,
            hasPeer: false,
            isStarted: false,
            pc:null,
            remoteStreamIndex: 0,
            pcConfig: {
              iceServers: []
            },
        }
    },
    mounted(){
        this.room = this.$route.params.room || prompt("[User] Room을 입력해주세요.(ex. agent ID)", "");
        this.$asocket.emit('hi', this.room, 'AR');
        this.$asocket.emit('ice');
        
        // iceserver receiving
        this.$asocket.on('ice', iceServers => {
          this.pcConfig.iceServers=iceServers;
          console.log('ice was loaded from signal server : ' + JSON.stringify(this.pcConfig.iceServers));
        });
        
        this.$asocket.on('joined', room => { 
            console.log('joined ' + room);
            this.joined = true;
        });

        this.$asocket.on('full', room => {
            console.log('Room ' + room + ' is full');
        });
        
        this.$asocket.on('leaved', room => {
            console.log('leaved ' + room);
            this.joined = false;
        });

        this.$asocket.on('peer joined', socketId => {
            console.log('peer joined : ' + socketId);
            if (!this.pcConfig.iceServers[0]) {
                console.log('ice server is not set yet');
                return
            }
            this.createPeerConnection();
            //this.sendMessage('ready monitoring audio');
            
        });
        
        this.$asocket.on('peer leaved', socketId => {
            console.log('audio sender leaved : ' + socketId);
            this.hasPeer = false;
            
        });

        this.$asocket.on('log', array => {
            console.log.apply(console, array);
        });

        // This client receives a message
        this.$asocket.on('message', (message, socketId) => {
            console.log('Client received message:', message);
            
            if (message.type === 'offer' && this.hasPeer && !this.isStarted) {
                if (!this.hasPeer) {
                    this.createPeerConnection();
                }
                this.pc.setRemoteDescription(new RTCSessionDescription(message));
                this.doAnswer();
                this.isStarted = true;
                
            } else if (message.type === 'candidate' && this.hasPeer) {
                var candidate = new RTCIceCandidate({
                    sdpMLineIndex: message.label,
                    candidate: message.candidate
                });
                this.pc.addIceCandidate(candidate);
            } else if(message === 'ready remote audio'){
                if (this.joined && this.hasPeer) {
                    if (this.isStarted) {
                         console.log('Im already started');
                    } else {
                        this.resetMedia();
                        this.sendMessage('ready monitoring audio');
                    }
                }
            } else if (message === 'bye remote audio' && this.hasPeer) {
                this.handleRemoteHangup();
            }
        });
        window.onbeforeunload = () => {
	  this.sendMessage("bye monitoring audio");
	  this.$asocket.emit('bye', this.room, 'AR');
	};
    },
    methods: {
        /*    common    */
        sendMessage(message) {
            console.log('Client sending message: ', message);
            this.$asocket.emit('message', message);
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
                console.log('Failed to create PeerConnection, exception: ' + e.message);
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
            
            
            if (mediaKind == 'audio') {
                var audio = document.createElement("audio");
                audio.id = 'remote-audio' + this.remoteStreamIndex++;
                
                var audios = document.querySelector("#remote-audios");
                audios.appendChild(audio);
                audio.srcObject = event.stream;
                audio.controls = true;
                audio.autoplay = true;
                
            }

        },
        handleRemoteStreamRemoved(event) {
            console.log('Remote stream removed. Event: ', event);
            
        },
        handleRemoteHangup() {
            this.stop();
            this.hasPeer = false;
        },
        resetMedia(){
            this.remoteStreamIndex = 0;
            console.log("Peerconnection close");
            //this.pc = null;
            //this.hasPeer = false;
        }



    }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#media {
  display: block;
  flex: 1;
  font-size: 0;
}

.remote-audios {
  position: absolute;
  width: 100%;
  height: 100%;
  object-fit: fill;
  outline: none;
}
</style>

