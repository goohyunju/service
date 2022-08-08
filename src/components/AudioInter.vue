<template>
  <div id="media">
    <h1>Audio Sending...</h1>
    <audio id="calling-audio"></audio>
    <div id="remote-audios"></div>
  </div>
</template>

<script>
import Vue from 'vue'

Vue.prototype.eventBus = new Vue();

export default {
  name: 'AudioInter',
  props: {},
  data() {
    return {
      room: "",
      hasPeer: false,
      remotes: {}, // {socketId:{isStarted:bool, pc:RTCPeerConnection}}
      localStreams: new Array(),
      mediaCount: 0,
      pcConfig: {
        iceServers: []
      },
      localAudioStream: null,
    }
  },
  watch: {},
  mounted() {
    
        // variable initialization
        this.room = this.$route.params.room || prompt("[User] Room을 입력해주세요.(ex. agent ID)", "");
        console.log('This client ID is : ' + this.$asocket.id);
	if (this.room !== '') {
	  this.$asocket.emit('hi', this.room, 'AS');
	  console.log('Attempted to create or  join room', this.room);
	}
        // iceserver receiving
        this.$asocket.emit('ice');
        this.$asocket.on('ice', iceServers => {
          this.pcConfig.iceServers = iceServers;
          console.log('ice was loaded from signal server : ' + JSON.stringify(this.pcConfig.iceServers));
        });
        
        /* enroll socket listening */
        this.$asocket.on('joined', room => {
            console.log('joined: ' + room);
            this.joined = true;
        });
        this.$asocket.on('full', room => {
            console.log('Room ' + room + ' is full');
        });
        this.$asocket.on('leaved', room => {
            console.log('leaved: ' + room);
            this.joined = false;
        });
        
        /* Another peer on the same room */
        this.$asocket.on('peer joined', socketId => {
            console.log('Peer ' + socketId + ' joined room ' + this.room);
            this.hasPeer = true;
            this.createPeerConnection(socketId);
            
            // localStreams에 저장된 스트림이 media 갯수와 같다면 send Message
	    console.log(this.localStreams.length, this.mediaCount);
            if (this.localStreams.length > 0 && (this.localStreams.length == this.mediaCount)){
	        this.sendMessage('ready remote audio');
            }
        });
        
        this.$asocket.on('peer leaved', socketId => {
            console.log('Peer ' + socketId + ' leaved room ' + this.room);
            delete this.remotes[socketId];
            
        });
	

        this.$asocket.on('log', array => {
            console.log.apply(console, array);
        });
        
        /* Message from receiver handler */
        this.$asocket.on('message', (message, socketId) => {
            if (socketId !== this.$asocket.id){
                console.log('Client received message:', message);
            }
            
            if (message === 'ready monitoring audio') {
                this.maybeStart(socketId);
            } else if (message.type === 'answer' && this.remotes[socketId].isStarted) {
                  this.remotes[socketId].pc.setRemoteDescription(new RTCSessionDescription(message))
            } else if (message.type === 'candidate' && this.remotes[socketId].isStarted) {
              var candidate = new RTCIceCandidate({
                sdpMLineIndex: message.label,
                candidate: message.candidate
              });
              this.remotes[socketId].pc.addIceCandidate(candidate);
            } else if (message === 'bye monitoring audio') {
                this.stop(socketId);
		console.log("remove stream for " + socketId);
                this.removePeerConnection(socketId);
		console.log("remove PeerConnection for " + socketId);
            } else if (message === 'reload'){
                window.location.reload();
            }
        });

        this.getConnectedDevices("audioinput").then((microphone) => {
          microphone.forEach((mic, index) => {
            this.openMic(mic.deviceId, mic.label).then(stream => {
              this.mediaCount += 1;
              this.localStreams.push(stream);
            }).catch((e) => {
              console.error('getUserMedia() error: ' + e.name);
            });
          });

        });
        
        window.onbeforeunload = this.hangup
  },
  methods: {
    async getConnectedDevices(type) {
      const devices = await navigator.mediaDevices.enumerateDevices();
      console.log("getConnectedDevices ", devices);
      return devices.filter(device => device.kind === type);
    },
    async openMic(micId, micLabel) {
      const constraints = {
        'audio': {
          'deviceId': micId
        },
        'video': false
      }
      return await navigator.mediaDevices.getUserMedia(constraints);
    },
    sendMessage(message) {
      console.log('Client sending message: ', message);
      this.$asocket.emit('message', message);
    },
    maybeStart(socketId) {
      console.log('>>>>>>> maybeStart() ', this.hasPeer);
      if (this.hasPeer && this.remotes[socketId]) {
        if (!this.remotes[socketId].isStarted) {
          console.log('>>>>>> creating peer connection');
          this.localStreams.forEach(stream => {
            //this.remotes[socketId].pc.addStream(stream);
            stream.getTracks().forEach((track) => {
                this.remotes[socketId].pc.addTrack(track, stream);
            });
          });
          this.remotes[socketId].isStarted = true;
        }
        this.doCall(socketId);
      } else {
        console.log('Channel is not ready yet');
      }      
    },

    createPeerConnection(socketId) {
      try {
        this.remotes[socketId] = {pc:new RTCPeerConnection(this.pcConfig), isStarted: false};
        this.remotes[socketId].pc.onicecandidate = this.handleIceCandidate;
        console.log('Created RTCPeerConnnection');
      } catch (e) {
        console.log('Failed to create PeerConnection, exception: ' + e.message);
        alert('Cannot create RTCPeerConnection object.');
        return;
      }
    },
    removePeerConnection(socketId) {
            try {
                delete this.remotes[socketId];
                console.log('Removed RTCPeerConnection');
            } catch (e) {
                console.error('Failed to remove PeerConnection, exception : ' + e.message);
                return
            }
        },
    handleIceCandidate(event) {
      console.log('icecandidate event: ', event);
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
    handleCreateOfferError(event) {
      console.log('createOffer() error: ', event);
    },
    doCall(socketId) {
	  console.log('Sending offer to peer');
	  this.remotes[socketId].pc.createOffer((sessionDescription) => {
	      this.remotes[socketId].pc.setLocalDescription(sessionDescription);
	      console.log('setLocalAndSendMessage sending message', sessionDescription);
	      this.sendMessage(sessionDescription);
	  }, this.handleCreateOfferError);
    },
    setLocalAndSendMessage(sessionDescription) {
      this.remotes[socketId].pc.setLocalDescription(sessionDescription);
      console.log('setLocalAndSendMessage sending message', sessionDescription);
      this.sendMessage(sessionDescription);
    },
    onCreateSessionDescriptionError(error) {
      console.log('Failed to create session description: ' + error.toString());
    },
    hangup() {
      console.log('Hanging up.');
      this.sendMessage("bye remote audio");
      this.$asocket.emit('bye', this.room, 'AS');
    },
    stop(socketId) {
      this.localStreams.forEach(stream => {
		this.remotes[socketId].pc.removeStream(stream);
	    });
      this.remotes[socketId].pc.close();
      this.remotes[socketId].isStarted = false;
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#media {
  display: block;
  flex: 1;
  font-size: 10;
}

#calling-audio div {
  display: inline-block;
  position: relative;
  width: 50%;
  height: 50%;
}
</style>

