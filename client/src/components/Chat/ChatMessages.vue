<template>
  <div> 
    <ul ref="ul" class="chat-messages" v-chat-scroll id="capture"> <!-- capture Must stay here for full chat record image + CLEAR CHAT -->
      <li  ref="chat-message" v-for="message in messages" :key="message.id" :class="{'message right-message': name === message.name, 'message left-message': name !== message.name}">
      <span class="message-avatar"
        :style='
          [ name !== message.name ?
            { backgroundImage: "url(" + "https://image.flaticon.com/icons/svg/547/547420.svg" + ")"} :
            { backgroundImage: "url(" + "https://image.flaticon.com/icons/svg/1596/1596810.svg" + ")"}
          ]'>
      </span>
      <div class="message-bubble">
        <div class="message-info">
          <span v-if="name !== message.name" class="message-info-name">{{ message.name }}</span>
        </div>
                  <!-- :style='
          [ message.embed ?
            { maxWidth: "100%", width: "350px"} :
            { maxWidth: "180px"}
          ]' -->
        <div class="message-text" 

        
        
        >

          <span v-if="message.embed">
            <div class="video-container" ref="parent">
              <!-- <YouSyncMessage :leID="message.embed" :messageid="message.id"/> -->
              <!-- <iframe width="560" height="315" loading="lazy"
                :src=" 'https://www.youtube.com/embed/' + message.embed" 
                frameborder="0" 
                allowfullscreen>
              </iframe>  TURNED OFF FOR NOW-->
              <button :id="message.embed" class="btn-small" style="position:absolute;bottom:0;" @click="sendID($event)">+</button>
            </div>
          </span>

          {{ message.content }}   
          
          <span class="message-info-time">{{ message.timestamp}}</span></div>

      </div>
      </li>
    </ul>

    <div id="youtube" class="displayNone" ref="ytbPanel">

      <div id="mainView">
      
     
        <div class="col1">
            <youtube id="playCss"
                ref="youtube"
                :video-id="stored" 
                :player-vars="playerVars"
                @ready="ready" 
                @playing="playing" 
                @paused="paused"
                @ended="ended"
            ></youtube>
          
            <div id="buttons"> 
                <button :id="stored" @click="playAll($event)">Play</button>
                <button :id="stored" @click="pauseAll($event)">Pause</button>
                <button :id="stored" @click="seekOnOthers($event)">Sync</button>
                <button :id="stored" @click="backToStart($event)">Restart</button>
                        
                <button :id="stored" @click="muteAll($event)">Mute</button>
                <button :id="stored" @click="unmuteAll($event)">Unmute</button>
                <button :id="stored" @click="startTheShow($event)">Show</button>
                <button :id="stored" @click="stopTheShow($event)">StopShow</button>
            </div>   

        </div>

        <div class="col2">
          <div id="youtubeLogs">
            <div id="bar"> 
              <!-- <span id="whatfor">Logs</span> -->
              
              <div id="red"></div>
              <div id="yellow"></div>
              <div id="green"></div>
            </div>
            <ul class="list-container" v-chat-scroll > 
              <li 
                  v-for="(event, i) in events" :key="`${i}-${event.id}`"
                  ref="playingPaused"
              
              > 
                <span class="id">{{event.id}} </span>  <!-- .substring(0, 8)-->
                <span :class="{'play': event.action.includes('playing'), 
                  'pause': event.action.includes('paused'), 
                  '': event.action == 'joined room.', 
                  'leftRoom': event.action == 'left room.',
                  'endView' : event.action == 'ended watching.',
                          
                          }"
                  > {{event.action}}
                </span>   

                          <a style="color: lightblue" 
                          class="videoid" 
                          v-if="event.videoid" 
                          :href="'https://www.youtube.com/watch?v='+event.videoid"
                          target="_blank"
                          rel="noopener noreferrer"
                          >link</a>

                          <span v-if="event.currentTime">{{event.currentTime}} </span>
                          <!--<span v-if="event.playCurrentTime">{{event.playCurrentTime}} </span> -->
          
                <span v-if="event.senderCurrentTime">{{event.senderCurrentTime}} </span>
                <!-- <span ref="fromNow" >{{ event.timestamp  }}</span>   Future reference, or local component-->
                <dynamic-from-now class="timestamp" :interval="60000"></dynamic-from-now>
              </li>
            </ul>
            <!--<button id="clear" >Delete</button> <! @click="clearLogs"-->
          </div>
        </div>
    


        <span ref="alert" id="alert" :v-if="this.alert === 'Resyncing. Clients not in sync.'" >{{this.alert}} </span>
      
      </div>
      
    </div>

    <div id="ytbSrch" ref="youtubeSearch" class="displayNone">
          <transition name="bounce">  
              <YoutubeSearch @sendToParent="finallyGetId"/>
        </transition>
      </div>

  
  </div>
</template>

<script>
/*eslint-disable*/
  import db from '@/firebase/init'
  import moment from 'moment'
  // import YouSyncMessage from '@/components/YouSyncMessage'
  import io from 'socket.io-client'; 
  import YoutubeSearch from '@/components/YoutubeSearch'
 

  export default {
    name: 'ChatBody',
    props: ['name', 'room'],
    components: {
      YoutubeSearch,

    },
    data(){
      return {
        messages: [],
        stored: "",
        socket: {},
        playerVars: {
            'rel': 0,
            // 'controls': 0
        },
        context: 0,
        position :{x: 0, y: 0},
        videoId: 'nx3egm8xIo8',
        events: [],
        username: "",
        youtubeId: "bwmSjveL3Lc", //2S24-y0Ij3Y
        randomkpop:[],
        currentTime: "",
        alert: "",
        state: "",
        personalTime: "",
        difference: 0,
        check: false,
        repetitive: null,
      
      }
    },
    created(){
      this.socket = io("http://localhost:3000/"); // http://192.168.100.3:3000/" "http://localhost:3000/" Client socket to > server adress / Gitpod change 

      let ref = db.collection('messages').orderBy('timestamp')
      ref.onSnapshot(snapshot => {
        snapshot.docChanges().forEach( change => {
          if(change.type == 'added'){
            let doc = change.doc
            this.messages.push({
              id: doc.id,
              name:doc.data().name,
              content: doc.data().content,
              timestamp: moment(doc.data().timestamp).fromNow(),
              room: doc.data().room,
              currentUser: this.name,
              embed: doc.data().embed
            })
          }
        })
      })
    },
    mounted(){

			this.socket.on('getCurrentTime', data => {  

				//Restul 3 primesc asta
                //let targetId = data.targetId;
				// #### THis is the magical formula i`ve been waiting to develop ! It finally compares both times without delay! ##############
				this.$refs.youtube.player.getCurrentTime().then(value => {
                            
                        this.difference = data.currentTime - value

						if (value < data.currentTime - 0.150 || value > data.currentTime + 0.150) {
						
							//this.player.seekTo(data.currentTime, true); // might need to time it out by .400 or so
						
							//this.$refs.syncWithMe.click();
							
								//this.player.seekTo(data.currentTime, true); - to auto sync
							

							// // Forces video to play right after seek
							//
							//console.log('They are not in .2 sync')
							this.alert= "🟡" + parseFloat(this.difference.toFixed(3))
						}
						
						else {
							//console.log('Clients are in sync')
							//Clients in sync
							this.alert= "🟢" + parseFloat(this.difference.toFixed(3))
						}
				});
					

		
				//this.events.push(data);
			})

			this.socket.on('seekOnOthers', data => {  
				 //1. Write to dom - Check
				//this.events.push(data)
				//console.log('Received back from server:' + " "+ data.senderCurrentTime)
				this.$refs.youtube.player.seekTo(data.senderCurrentTime, true); // Basically saying this ytb component with ref of id seek
				
				//All the logic should be done server side

			})
				
			this.socket.on('pause_all', data => {  
				//this.events.push(data);
				this.$refs.youtube.player.pauseVideo();
            })
            
			this.socket.on('play_all', data => {  
                this.stored = data.targetId;
                //this.events.push(data);

              //You need to delay it or it won't happen
              setTimeout(()=>{
                     this.$refs.ytbPanel.classList.remove('displayNone')
                    this.$refs.ul.classList.add('shortenBox')
                this.player.playVideo();
              },400)
                   
             
          
          


         // Hitting play button only on the id that generated it
                // }
          //need to change the videoId here
        })

        this.socket.on('backToStart_all', data => {  

                //this.events.push(data);
				this.$refs.youtube.player.seekTo(0, true)	
			})
      this.socket.on('playing', data => {  
				//this.state = data.id + " is " + data.action;

				console.log("This state is now :" + this.state ) //Works
				//console.log(data.action + " " + data.id) //Works
				this.state = data.id + " is " + data.action + " at "+ data.currentTime;
				
				this.events.push(data);
			})

			this.socket.on('paused', data => {  

				this.state = data.id + " is " + data.action;
				this.events.push(data);		
			})

			//Shutting this off for it is annoying atm
			//var username = prompt('What\'s your username?');

			//if (username){ 

				// this.socket.emit('little_newbie', username);

				// Sectia primire de catre Client inapoi de la server



			this.socket.on('disconnect', data => {  
				this.events.push(data);
			})


			this.socket.on('ready', data => {  
					this.events.push(data); //write to array, which will output to dom with v-for
			})
			this.socket.on('ended', data => {  
					this.events.push(data); //write to array, which will output to dom with v-for
			})



			this.socket.on('mute_all', data => {  
			
				this.player.mute()
				this.events.push(data);
			})
			this.socket.on('unmute_all', data => { 
				this.player.unMute()
				this.events.push(data);
			})
			this.socket.on('changeSong_all', data => {  

				//console.log( 'This comes from mounted' + this.youtubeId)	
				//So this socket gets but not all.. the pbolem is in Socketio emit!
				this.player.loadVideoById(data.videoid, 0, "large")
				this.events.push(data);
				//console.log(data.youtubeId); This check is OK!
		
			})


			// this.socket.on('little_newbie', data => {  
			// 	this.events.push(data); //write to array, which will output to dom with v-for
			// })

			//}

			//setInterval(this.getNow, 5000);//refs are available only after mounted

    },
    computed: {
      player() {
          return this.$refs.youtube.player
      }
    },
    methods: {

      finallyGetId(params){
      this.stored = params; // Finally, params are coming from VideoGridItem > SearchResults >YoutubeSearch> HERE . Vuex overrated! ;)
      },
      seekOnOthers(event){
                  // Get Sender current time and pass it along seekOnOthers
                  let targetId = event.currentTarget.id;

        this.$refs.youtube.player.getCurrentTime().then(value => {
          //console.log('The one i clicked seek button is on:'+value)
          this.socket.emit("seekOnOthers", value, targetId)
        });
        
              },
      playAll(event){
        let targetId = event.currentTarget.id; //Needs to be outside here
        //console.log(targetId + "initiated play");
              
        this.$refs.youtube.player.getCurrentTime().then(value => {
          // Do something with the value here
                      //console.log('I paused at '+ value)
                      
          this.socket.emit("play_all", value, targetId  )
        });
      },
      pauseAll(event){
        let targetId = event.currentTarget.id;

        this.$refs.youtube.player.getCurrentTime().then(value => {
          // Do something with the value here
                      //console.log('I paused at '+ value)
                    
          this.socket.emit("pause_all", value, targetId)
        });
              },
              backToStart(event){
                  let targetId = event.currentTarget.id;
        this.socket.emit("backToStart_all", targetId)
      },
              startTheShow(event){
                  
                  // console.log(targetId+ "from outside") checked
                    let targetId = event.currentTarget.id;
                    this.repetitive = window.setInterval(() => {
                      //  console.log(targetId + "from inside setInterval") chcked

          this.$refs.youtube.player.getCurrentTime().then(value => {
            // Do something with the value here
            //console.log(value)
            
            this.socket.emit("getCurrentTime", value, targetId)
            
          });


          }, 1500)
              },
              stopTheShow(){
                  window.clearInterval(this.repetitive);
              },

      ready (event) { //This guys tells me state of player, OH it shouts automatically
          this.player = event.target;
          console.log('Player is ready.')
          this.socket.emit("ready");
      },
      ended (){
        console.log('Yay. You`ve stayed until the end . Video ended!')
        this.socket.emit("ended");
      }, 
      playVideo() {
        this.player.playVideo()
      },
      pauseVideo(){
        this.player.pauseVideo()
      },
      // seekTo(){
      // 	this.player.seekTo(5 , true)
      // },

      muteAll(){
        this.socket.emit("mute_all")
      },
      unmuteAll(){
        this.socket.emit("unmute_all")
      },
      changeSong(){
        // console.log(this.youtubeId); OK check
        
        // TO DO - Determine if playlist and play it all!

        //Determine if typed a full link or just ID
        if  (this.youtubeId.includes("www.youtube.com/watch")){
          this.socket.emit("changeSong_all", this.$youtube.getIdFromUrl(this.youtubeId))
          //this.$refs.youtubeIdInput.value="";
        } else {
          this.socket.emit("changeSong_all", this.youtubeId)
          //this.$refs.youtubeIdInput.value="";
        }

        
          
          //console.log(this.youtubeId)	
          //So it can access data () with this
      },

      // All clients call these automatically when the API itself detects change

      playing () {

      	// Don't initialize state here, but on server and receive ;)
      	this.state = "playing"

      	this.player.getCurrentTime().then(value => {
      		// Do something with the value here
      		//console.log('See'+ value)
          
      		this.socket.emit("playing", value)
      	});
            
      },
      paused () {
        
      	this.player.getCurrentTime().then(value => {
      		// Do something with the value here
      		//console.log('I paused at '+ value)
      		this.socket.emit("paused", value)
      	});

      },

      // buffering (){
      // 	//this.value = this.player.getPlayerState()
        
      // 	this.socket.emit("buffering");
      // }, 
      getNotifications(){
        if (this.state !== "")
        console.log(this.state)
      },
      sendID(event){
        this.stored = event.target.id;
        //console.log(event.target.id)
      },


      },
  }
</script>

<style lang="css">

  .bounce-enter-active {
  animation: bounce-in .5s;
}
.bounce-leave-active {
  animation: bounce-in .5s reverse;
}
@keyframes bounce-in {
  0% {
    transform: scale(0);
  }
  50% {
    transform: scale(1.5);
  }
  100% {
    transform: scale(1);
  }
}



  #ytbSrch{
    /* position:fixed;
    bottom:100px; */
    max-height:250px;
    width:100%;
    max-width:550px;
    background:orange;
    overflow-y:auto;
  }
  #ytbSrch h3{
    font-size:120%;
  }

  #mainView iframe {
    width: 100%;
    /* max-width:100%; */
    /* margin:0 auto; Dont want it centered now */

     display:block; 


  }
  #mainView #buttons {
    max-width:338px;
    /* margin:0 auto; Dont want it centered now */
     display:block; 

  }
    #buttons button{
      background-color:lightseagreen;
      font-size:10.95px;
     
      /* margin-top:150px; */
      
  }

  .displayNone{
    display:none;
    
  }
  .video-container {
  position: relative;
  padding-bottom: 56.25%;
  padding-top: 30px; height: 0; overflow: hidden;
  }
  
  .video-container iframe,
  .video-container object,
  .video-container embed {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }
   #youtube{
     /* width:100%;
     max-width:100%; */
    /* height:226px; 
    background:chocolate; */
    /* width:100%; */
    padding:5px;
    transition: all .5s ease-in-out;
  }


  /*.chat-messages*/
  .chat-messages {
   
    /* transform: scale(1,-1); */
    /* padding-top:2px;
    padding-bottom:2px; */
    z-index:1;
    overflow-y: auto;
    padding: 10px;
    background-color: #fcfcfe;
    height: calc(100vh - 153px);
     /*Ca sa dea exact minus header si new message*/ 

    /*Container Control
    outline: 1px solid red !important;
    outline-offset: -1px;*/
  }

  .chat-messages::-webkit-scrollbar {
    width: 6px;
  }
  .chat-messages::-webkit-scrollbar-track {
    background: #ddd;
  }
  .chat-messages::-webkit-scrollbar-thumb {
    background: #bdbdbd;
  }
  /*.messages*/
  .messages {
    padding: 3px;
  }
  .messages-box {
    overflow: auto;
    visibility: hidden;
  }
  .messages, .messages-box:hover, .messages-box:focus {
    visibility: visible;
  }
  .messages-box::-webkit-scrollbar{
    width: 12px;
    background-color: #134446;
  }
  .messages-box::-webkit-scrollbar-track{
    -webkit-box-shadow: inset 0 0 6px rgba(0,0,0,0.3);
    border-radius: 10px;
    background-color: #F5F5F5;
  }
  .messages-box::-webkit-scrollbar-thumb{
      border-radius: 10px;
    -webkit-box-shadow: inset 0 0 6px rgba(0,0,0,.3);
    background-color: #009688;
  }
  /*message*/
  .message {
    display: flex;
    align-items: flex-end;
    margin-bottom: 10px;
  }
  .message:last-of-type {
    margin: 0;
  }
  .message-bubble {
    max-width: 450px;
    padding: 8px;
    border-radius: 15px;
    background: #ececec;
  }
  .message-info {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 5px;
  }
  .message-info-name {
    margin-right: 10px;
    font-weight: bold;
    font-size:18px;
  }
  .message-info-time {
    font-size: 0.85em;
    opacity:0.6;
    float:right;
    margin-left:3px;
  }
  .left-message .message-bubble {
    border-bottom-left-radius: 0;
  }
  .right-message {
    flex-direction: row-reverse;
  }
   .right-message .message-avatar {
    margin: 0 0 0 10px;
  }
  .right-message .message-bubble {
    background: #044C62;
    color: #FFFCFF;
    border-bottom-right-radius: 0;
  }
  .message-text{
    max-width:180px;
  }
  .message-avatar {
    width: 50px;
    height: 50px;
    margin-right: 10px;
    background: #ddd;
    background-repeat: no-repeat;
    background-position: center;
    background-size: cover;
    border-radius: 50%;
    margin-left: 5px;
  }
 

  
  #main-view{
      /* position:relative; */
      
  }

  /*Responsive Yt Embeds*/ 
  .video-container {
      position: relative;
      padding-bottom: 56.25%;
      padding-top: 30px; height: 0; overflow: hidden;
  }

  .video-container iframe,
  .video-container object,
  .video-container embed {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  }

  #alert{
      position: absolute;
      z-index: 99999;
      top: 2px;
      right:2px;
      /* background: white;
      border: 1px solid orange; */
      color:black;
      background:white;
      opacity:0.6;
      width:55px;
      font-size:11px;
  }
  #youtubeLogs{
  


  }
  #YouSync{
      max-width:426px;
      width:100%;
  }
  #playCss{
     
      /* display:none; */
      width:100%;
      max-height:190px;
      max-width:338px;
      
  }




  /*MacOs Terminal*/ 

	#bar {
     width:100%;
			text-align: center;
			/* width: 424px; */
			height: 20px;
			background-color: rgb(156, 143, 117);
			margin: 0 ;
			font-family: monospace;
			padding: auto;
			float: none;
			border-radius: 5px 5px 0px 0px ;
			position: relative;
      padding:3px;
	}
	#red {
			background-color: #E94B35;
			border-radius: 100%;
			width: 15px;
			height: 15px;
      float:right;
      opacity:0.6;
		  margin-left:5px;
	}
	#yellow {
			background-color: #f0f000;
			border-radius: 100%;
			width: 15px;
			height: 15px;
		 float:right;
			/* left: -180px;
			bottom: 40%; */
		 opacity:0.6;
		 margin-left:5px;
	}
	#green {
			background-color: #1AAF5C;
			border-radius: 100%;
			width: 15px;
			height: 15px;
			/* margin: 0 auto; */
			/* left: -160px;
			bottom: 99%; */
	    float:right;
		 opacity:0.6;
     margin-left:5px;
	}

  ul.list-container  {
      list-style-type: none;
      font-size: 11px;
      height: 150px;
      /* width:400px; */
      margin:0;
      overflow-y: auto;
      background-color: #33485E;
      color:#ffffff;
      padding: 4px 8px 8px 8px;
      border-radius: 0px 0px 8px 8px;
      min-height: 194px;
  }
 
   .timestamp{
      opacity: 0.75;
      font-size:9px;
      color: #ffffff;
      font-weight: 400;
      padding-left:3px;
      float: right;
  }

  #clear{
      float: right;
  }

  .play{
      color: #00ff00;
  }
  .pause{
          color: #f1f227;
  }
  .id {
      color: #be90d4;
  }
  .leftRoom{
      text-decoration: line-through;

  }
  .endView{
      color: #ff6347
  }
  .col1{
    float:left;
    max-width: 338px;
  }
  .col2{
    float:left;
    width: calc(100% - 338px);
    padding-left:2px;
  }
  #mainView:after {
    content: "";
    display: table;
    clear: both;
  }

</style>