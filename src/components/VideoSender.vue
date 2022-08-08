<template>
  <div id="media">
    <h1>Video Sending...</h1>
    <div id="videos"></div>
  </div>
</template>

<script>
import Vue from 'vue'

Vue.prototype.eventBus = new Vue();

export default {
    name: 'VideoSender',
    props:{
    },
    data(){
        return{
            room: "",
            joined: false,
            hasPeer: false,
            remotes:{}, // {socketId:{isStarted:bool, pc:RTCPeerConnection}}
            localStreams: new Array(),
            mediaCount: 0,
            pcConfig: {
              iceServers: []
            }
        }
    },
    watch:{
    },
    mounted(){
        // variable initialization
        this.room = this.$route.params.room || prompt("[User] Room을 입력해주세요.(ex. agent ID)", "");
        console.log('This client ID is : ' + this.$vsocket.id);
	if (this.room !== '') {
	  this.$vsocket.emit('hi', this.room, 'VS');
	  console.log('Attempted to create or  join room', this.room);
	}
        // iceserver receiving
        this.$vsocket.emit('ice');
        this.$vsocket.on('ice', iceServers => {
          this.pcConfig.iceServers = iceServers;
          console.log('ice was loaded from signal server : ' + JSON.stringify(this.pcConfig.iceServers));
        });
        
        /* enroll socket listening */
        this.$vsocket.on('joined', room => {
            console.log('joined: ' + room);
            this.joined = true;
        });
        this.$vsocket.on('full', room => {
            console.log('Room ' + room + ' is full');
        });
        this.$vsocket.on('leaved', room => {
            console.log('leaved: ' + room);
            this.joined = false;
        });

        /* Another peer on the same room */
        this.$vsocket.on('peer joined', socketId => {
            console.log('Peer ' + socketId + ' joined room ' + this.room);
            this.hasPeer = true;
            this.createPeerConnection(socketId);
            
            // localStreams에 저장된 스트림이 camera 갯수와 같다면 send Message
	    console.log(this.localStreams.length, this.mediaCount);
            if (this.localStreams.length > 0 && (this.localStreams.length == this.mediaCount)){
	        this.sendMessage('ready remote video');
            }
        });
        
        this.$vsocket.on('peer leaved', socketId => {
            console.log('Peer ' + socketId + ' leaved room ' + this.room);
            delete this.remotes[socketId];
            
        });
	

        this.$vsocket.on('log', array => {
            console.log.apply(console, array);
        });

        this.$vsocket.on('message', (message, socketId) => {
            if (socketId !== this.$vsocket.id){
                console.log('Client received message:', message);
            }
            
            if (message === 'ready monitoring video') {
                this.maybeStart(socketId);
            } else if (message.type === 'answer' && this.remotes[socketId].isStarted) {
                  this.remotes[socketId].pc.setRemoteDescription(new RTCSessionDescription(message))
            } else if (message.type === 'candidate' && this.remotes[socketId].isStarted) {
              var candidate = new RTCIceCandidate({
                sdpMLineIndex: message.label,
                candidate: message.candidate
              });
              this.remotes[socketId].pc.addIceCandidate(candidate);
            } else if (message === 'bye monitoring video') {
                this.stop(socketId);
		console.log("remove stream for " + socketId);
                this.removePeerConnection(socketId);
		console.log("remove PeerConnection for " + socketId);
            } else if (message === 'reload'){
              window.location.reload();
            }
        });
        
        // Get input of video and audio
        this.getConnectedDevices('videoinput').then((cameras) => {
	  if(cameras && cameras.length > 0) {
	    cameras.forEach((camera,index) => {
		this.openCamera(camera.deviceId,camera.label,323,720).then((stream) => {
		    this.mediaCount += 1;
		    this.addLocalStream(index, stream);
		}).catch((e) => {
		    console.error('openCarmera() Failed : ' + e);
		})
	    });
	    
	  }
	  console.log('Cameras were added to local streams. Count : ' + this.mediaCount);

        });
        
	
    },
    beforeUnmount() {
        this.hangup()
    },
    methods: {
        /*    common    */
        sendMessage(message) {
            console.log('Client sending message: ', message);
            this.$vsocket.emit('message', message);
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
        /*    VS specific    */
        createPeerConnection(socketId) {
            try {
                this.remotes[socketId] = {pc:new RTCPeerConnection(this.pcConfig), isStarted: false};
                this.remotes[socketId].pc.onicecandidate = this.handleIceCandidate;
                console.log('Created RTCPeerConnnection');
            } catch (e) {
                console.log('Failed to create PeerConnection, exception: ' + e.message);
                console.error('Cannot create RTCPeerConnection object.');
                return;
            }
        },
        removePeerConnection(socketId) {
            try {
                delete this.remotes[socketId];
            } catch (e) {
                console.error('Failed to remove Peerconnection : already removed');
            }
        },
        stop(socketId) {
            this.localStreams.forEach(stream => {
		this.remotes[socketId].pc.removeStream(stream);
	    });
            this.remotes[socketId].pc.close()
            this.remotes[socketId].isStarted = false;
        },
        async getConnectedDevices(type){
            const devices = await navigator.mediaDevices.enumerateDevices();
            console.log("getConnectedDevices ",devices);
            return devices.filter(device => device.kind === type);
        },
        async openCamera(cameraId, cameraLabel, minWidth, minHeight) {
	    const constraints = {
		'audio': false,
		'video': {
		  'deviceId': cameraId
		}
	    }
	    return await navigator.mediaDevices.getUserMedia(constraints);
	},
        
        addLocalStream(index, stream) {
	  console.log('Adding local stream.');

	  var video = document.createElement("video");
	  video.id = 'localVideo' + index;
	  video.autoplay = true;

	  var videos = document.getElementById("videos");
	  videos.appendChild(video);
	  video.srcObject = stream;
	  this.localStreams.push(stream);

	  
	},
	maybeStart(socketId) {
	    /* 
	    Peer Connection 이 아직 없으면 => X
	    시작 아직 안했으면 => localstream 을 추가한 후 offer
	    이미 시작해있으면 => offer
	    */
	  console.log('>>>>>>> maybeStart() ', this.hasPeer);
	  if (this.hasPeer && this.remotes[socketId])
	  {
	      if (!this.remotes[socketId].isStarted)
	      {
                console.log('>>>>>> creating peer connection');
		this.localStreams.forEach(stream => {
			this.remotes[socketId].pc.addStream(stream);
			console.log("stream info: ",stream);
		});
		this.remotes[socketId].isStarted = true;
	      }
	      this.doCall(socketId);
	  }
	  else
	  {
	      console.log('Channel is not ready yet');
	  }
	  
	  
	},
	doCall(socketId) {
	  console.log('Sending offer to peer');
	  this.remotes[socketId].pc.createOffer((sessionDescription) => {
	      this.remotes[socketId].pc.setLocalDescription(sessionDescription);
	      console.log('setLocalAndSendMessage sending message', sessionDescription);
	      this.sendMessage(sessionDescription);
	      }, this.handleCreateOfferError);
	},
        handleCreateOfferError(event) {
	  console.error('createOffer() error: ', event);
	},
	hangup() {
	  console.log('hangup');
	  this.sendMessage("bye remote video");
	  this.$vsocket.emit('bye', this.room, 'VS');
	},
        
    }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#media{
    display: block;
    flex: 1;
    font-size: 10;
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
