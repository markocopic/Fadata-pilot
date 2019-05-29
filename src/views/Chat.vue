<template>
<div class="chet">
    <v-container>
        <v-layout row wrap>
            
        
        <v-flex xs2>
            <h2 style="color:white;margin:20px 5px;">Participants:</h2>
            <p v-for="(partic,i) in chatParticipants" :key="i" style="color:white;margin:20px 5px;">
                {{partic.name}} <v-icon :color="partic.seen?'green':'red'">info</v-icon>
            </p>
        </v-flex>
        <v-flex xs8>
        <div id="chat" style="background:white; text-align:center;">

            <v-select
                style="width:300px;"
                :items="chatList"
                v-model="chat"
                label="select chat"
                class="d-inline-block pa-1"
                :disabled="loader"
            ></v-select>
            <v-btn :disabled="loader" class="d-inline-block" color="success" @click="openChat">open chat</v-btn>

            
            <v-select
                style="width:300px;"
                :items="peopleList"
                v-model="person"
                label="select person"
                class="d-inline-block pa-1"
                :disabled="loader"
            ></v-select>
            <v-btn :disabled="loader" class="d-inline-block" color="success" @click="invite">invite</v-btn>
            <v-btn :disabled="loader" class="d-inline-block" color="primary" @click="startNewChat">start new chat</v-btn>

            <v-text-field id="inputField" :disabled="disabled || loader" style="width:300px;" class="pa-2 ma-2 d-inline-block" @keyup.enter="send" placeholder="enter text" v-model="message"></v-text-field>
            <v-btn :disabled="loader" :loading="disabled" class="d-inline-block" color="success" @click="send">send</v-btn>


            <ul>
                <h1 style="color:white;">Channel: {{channel}}</h1>
                <li class="listItem" :class="msg.id=='SYSTEM'?'neutral':msg.id==userName?'greens':'yellows'" v-for="(msg,i) in messages" :key="i">
                    <p style="font-size:9px;margin:5px 0 0;">{{ msg.time }}</p>
                    <h2 class="mt-0">{{ msg.id }} </h2>
                    <p>{{ msg.text }}</p><hr class="grey">
                </li>
            </ul>
            
        </div>

        </v-flex>

        </v-layout>

    </v-container>

    <!-- LOADER -->
        <div class="text-xs-center">
            <v-dialog
            v-model="loader"
            hide-overlay
            persistent
            width="300"
            >
            <v-card
                color="primary"
                dark
            >
                <v-card-text>
                Joining chat channel
                <v-progress-linear
                    indeterminate
                    color="white"
                    class="mb-0"
                ></v-progress-linear>
                </v-card-text>
            </v-card>
            </v-dialog>
        </div>
  </div>
</template>

<script>

export default {
    watch:{
        '$route' (to, from){
            let self = this;
           this.messages = [];
           this.channel = to.params.id;
           if(!this.loader)
                this.loader = true;
           this.startNew(this.channel);
            setTimeout(() => {
                this.loader = false;
                setTimeout(() => {
                    document.getElementById("inputField").focus();
                    self.getParticipants();
                }, 500);
            }, 7000);

        }
        
    },
    data(){
        return{
            messages:[],
            message:null,
            userName: sessionStorage.getItem('name'),
            userId: sessionStorage.getItem('id'),
            channel:'',
            participants:{},
            disabled:false,
            baseURL: this.$store.state.baseURL,
            peopleList:[],
            person:null,
            chatList:[],
            chat:null,
            chatParticipants:[],
            openChannels:[],
            loader:true
        }
    },
    beforeDestroy(){
        let payload = [this.channel, this.userId, this.userName, btoa(sessionStorage.getItem('id'))];
        
        socket.emit('leave',payload);

        
    },
    mounted(){
        let self = this;
        this.channel = this.$route.params.id;
        //this.loadNewChat();
        this.startNew(this.channel)
        
        setTimeout(() => {
            self.loader = false;
            setTimeout(() => {
                document.getElementById("inputField").focus();
                self.getParticipants();
            }, 500);
        }, 9000);

        // socket.on('updateChat.' + this.channel, function(){
        //     //console.log('update chat', self.channel);
        //     setTimeout(() => {
        //         self.getParticipants();
        //     }, 1000);
        // });
    },
    methods:{
        getParticipants(){
            console.log('GETTING PARTICIPANTS');
            
            let self = this;
            let param = "cm:name:" + this.channel + "-invitations";
    

            // checking if there is already JSON file with that name
            axios.post(this.baseURL + 'search/versions/1/search',
                {
                "query":{
                    "query":param
                }
                },
                {
                headers:{
                    Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                }
            }).then(response =>{
                
                if(response.data.list.entries.length<1){
                    console.log('NO PARTICIPANTS');
                    //console.log(response.data);
                    
                    self.chatParticipants = [];
                }
                else{
                    let id = response.data.list.entries[0].entry.id;
                    
                    axios.get(this.baseURL + 'alfresco/versions/1/nodes/'+id+'/content',
                    {
                        headers:{
                        Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                        }
                    }).then(response => {
                        //console.log('chat participants', response.data);
                        self.chatParticipants = response.data;
                    
                    })
                }
                    
            }).catch(err => {
                console.log(err);
            })


        },
        startNewChat(){

            this.channel += Math.ceil(Math.random()*100);
            this.$router.push('/chat/' + this.channel);
        },
        openChat(){
            if(this.chat){

                this.$router.push('/chat/' + this.chat);
            }
                
        },
        startNew(chat){
            
            this.chatParticipants = [];
            
            let self = this;

            let user = [this.channel, this.userId, this.userName, btoa(sessionStorage.getItem('id'))];
        
            this.channel = chat;

            this.getParticipants();
            
            ///

                let param = 'cm:description:' + sessionStorage.getItem('name')
                axios.post(this.baseURL + 'search/versions/1/search',
                    {
                    "query":{
                        "query":param
                    }
                    },
                    {
                    headers:{
                        Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                    }
                }).then(response =>{
                    let list = response.data.list.entries;
                    for (let i = 0; i < list.length; i++) {
                        this.chatList.push(list[i].entry.name);
                    }
                    
                })

                axios.get(this.baseURL + 'alfresco/versions/1/people/',{
                headers:{
                            Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                        }
                })
                .then(response => {
                    //console.log(response.data);
                    this.peopleList = [];
                let list =  response.data.list.entries;
                for (let i = 0; i < list.length; i++) {
                    this.peopleList.push(list[i].entry.id)
                }
                
                
                })

            ///
            
            this.messages = [];

            let payload = [this.channel, this.userId, this.userName, btoa(sessionStorage.getItem('id'))];

            socket.emit('join',payload);

            let duplicate = false;

            for (let i = 0; i < this.openChannels.length; i++) {
                if(this.openChannels[i] == this.channel)
                    duplicate = true;
            }
            
            if(duplicate == false){
                socket.on('chat.' + self.channel, function(payload){
                    if(payload[0] != self.channel) return;
                    
                    let obj = {};
                    obj.id = payload[1];
                    obj.text = payload[2];
                    obj.time = payload[4];

                    self.messages.unshift(obj);
                    self.disabled = false;
                    console.log('NEW MESSAGE');
                    setTimeout(() => {
                        document.getElementById("inputField").focus();
                    }, 500);
                })

                socket.on('updateChat.' + this.channel, function(){
                   // console.log('update chat', self.channel);
                    setTimeout(() => {
                        self.getParticipants();
                    }, 1000);
                });

                this.openChannels.push(this.channel);
            }

            


            socket.on('prevchat.' + self.channel, function(payload){
                if(self.messages.length < 1){
                    for (let i = 0; i < payload.length; i++) {
                        let obj = {};
                        obj.id = payload[i].name;
                        obj.text = payload[i].text;
                        obj.time = payload[i].time;
                        self.messages.unshift(obj)
                    }
                }
            })

            socket.on('participants.' + self.channel, function(payload){
                //console.log(payload);
                self.participants = payload;
                //console.log(self.participants);
                
                
            })


        },
        saveInvitation(){
            let param = "cm:name:" + this.person + "-notifications";
    

            // checking if there is already JSON file with that name
            axios.post(this.baseURL + 'search/versions/1/search',
                {
                "query":{
                    "query":param
                }
                },
                {
                headers:{
                    Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                }
            }).then(response =>{
                
                if(response.data.list.entries.length<1){
                    console.log('NO NODE');
                    this.createNode();
                }
                else{
                    console.log('UPDATING NODE');
                    let id = response.data.list.entries[0].entry.id;
                    this.updateNode(id);
                }
                    
            }).catch(err => {
                console.log(err);
            })
        },
        createNode(){
            let self = this;
            let obj = {}
            obj.name = this.person + "-notifications";
            obj.nodeType = "cm:content";

            let result = [{ host: this.userName, channel: this.channel}];

            axios.post(this.baseURL + 'alfresco/versions/1/nodes/45dbad81-e657-4020-9266-e09dc597c25f/children?autoRename=true',
                JSON.stringify(obj),
                {
                    headers:{
                    Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                    }
                }).then(response => {
                    axios.put(self.baseURL + 'alfresco/versions/1/nodes/'+response.data.entry.id+'/content?majorVersion=false',
                    JSON.stringify(result),
                    {
                        headers:{
                        Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                        }
                    }).then(response => {
                       // console.log(response);
                        
                    })
                
                })
        },
        updateNode(id){
            let self = this;
            axios.get(this.baseURL + 'alfresco/versions/1/nodes/'+id+'/content',
                {
                    headers:{
                    Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                    }
                }).then(response => {
                    let duplicate = false;
                    let result = { host: this.userName, channel: this.channel};
                    let arr = response.data;
                    for (let i = 0; i < arr.length; i++) {
                        if(arr[i].channel == self.channel)
                            duplicate = true;
                    }
                    if(!duplicate)
                        arr.push(result);

                    axios.put(self.baseURL + 'alfresco/versions/1/nodes/'+id+'/content?majorVersion=false',
                    JSON.stringify(arr),
                    {
                        headers:{
                        Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                        }
                    }).then(response => {
                        //console.log(response);
                    })
                
                })
        },
        invite(){
            if(this.person){
                this.saveInvitation();
                this.saveStatus();
            }
        },
        send(){
            if(!this.message || this.message.trim() == '') {
               this.message = null; 
               return;
            }
            this.disabled = true;
            let payload = [this.channel, this.userName, this.message, btoa(sessionStorage.getItem('id')), new Date().toLocaleString()];

            socket.emit('chat', payload);

            this.message = null;
        },
        loadNewChat(){
        

    //     let param = 'cm:description:' + sessionStorage.getItem('name')
    //     axios.post(this.baseURL + 'search/versions/1/search',
    //      {
    //        "query":{
    //          "query":param
    //        }
    //      },
    //      {
    //        headers:{
    //          Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
    //        }
    //    }).then(response =>{
    //        let list = response.data.list.entries;
    //        for (let i = 0; i < list.length; i++) {
    //           this.chatList.push(list[i].entry.name);
    //        }
           
    //    })

    //     axios.get(this.baseURL + 'alfresco/versions/1/people/',{
    //       headers:{
    //                  Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
    //              }
    //     })
    //     .then(response => {
    //          //console.log(response.data);
    //         this.peopleList = [];
    //      let list =  response.data.list.entries;
    //       for (let i = 0; i < list.length; i++) {
    //           this.peopleList.push(list[i].entry.id)
    //       }
         
          
    //     })

        
    //     let self = this;

    //     let payload = [this.channel, this.userId, this.userName, btoa(sessionStorage.getItem('id'))];

    //     socket.emit('join',payload);

    //     socket.on('connect', function(){
    //         //socket.emit('join', payload);
    //         console.log('connected to server');
    //     })

    //     socket.on('chat.' + self.channel, function(payload){
    //         //console.log(payload);
    //         let obj = {};
    //         obj.id = payload[1];
    //         obj.text = payload[2];
    //         obj.time = payload[4];
    //         self.messages.unshift(obj);
    //         self.disabled = false;
    //         console.log('stop');
            
    //     })


    //     socket.on('prevchat.' + self.channel, function(payload){
    //         if(self.messages.length < 2){
    //              for (let i = 0; i < payload.length; i++) {
    //                 let obj = {};
    //                 obj.id = payload[i].name;
    //                 obj.text = payload[i].text;
    //                 obj.time = payload[i].time;
    //                 self.messages.unshift(obj)
    //             }
    //         }
    //     })

        // socket.on('participants.' + self.channel, function(payload){
        //     //console.log(payload);
        //     self.participants = payload;
        //     //console.log(self.participants);
            
            
        // })
        },
        saveStatus(){
            let param = "cm:name:" + this.channel + "-invitations";
    

            // checking if there is already JSON file with that name
            axios.post(this.baseURL + 'search/versions/1/search',
                {
                "query":{
                    "query":param
                }
                },
                {
                headers:{
                    Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                }
            }).then(response =>{
                
                if(response.data.list.entries.length<1){
                    console.log('NO INVITATION FILE');
                    this.createStatus();
                }
                else{
                    console.log('UPDATING INVITATION FILE');
                    let id = response.data.list.entries[0].entry.id;
                    this.updateStatus(id);
                }
                    
            }).catch(err => {
                console.log(err);
            })
        },
        createStatus(){
            let self = this;
            let obj = {}
            obj.name = this.channel + "-invitations";
            obj.nodeType = "cm:content";

            let result = [{ name:self.person, seen:false}];

            axios.post(this.baseURL + 'alfresco/versions/1/nodes/45dbad81-e657-4020-9266-e09dc597c25f/children?autoRename=true',
                JSON.stringify(obj),
                {
                    headers:{
                    Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                    }
                }).then(response => {
                    axios.put(self.baseURL + 'alfresco/versions/1/nodes/'+response.data.entry.id+'/content?majorVersion=false',
                    JSON.stringify(result),
                    {
                        headers:{
                        Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                        }
                    }).then(response => {
                        //console.log(response);
                        socket.emit('invite', self.person,self.channel, self.userName);
                    })
                
                })
        },
        updateStatus(id){
            let self = this;
            axios.get(this.baseURL + 'alfresco/versions/1/nodes/'+id+'/content',
                {
                    headers:{
                    Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                    }
                }).then(response => {
                    let duplicate = false;
                    let result = { name:self.person, seen:false };
                    let arr = response.data;
                    for (let i = 0; i < arr.length; i++) {
                        if(arr[i].name == self.person){
                            arr[i].seen = false;
                            duplicate = true;
                        }   
                    }
                    if(!duplicate)
                        arr.push(result);

                    axios.put(self.baseURL + 'alfresco/versions/1/nodes/'+id+'/content?majorVersion=false',
                    JSON.stringify(arr),
                    {
                        headers:{
                        Authorization: 'Basic ' + btoa(sessionStorage.getItem('id'))
                        }
                    }).then(response => {
                       // console.log(response);
                        socket.emit('invite', self.person,self.channel, self.userName);
                    })
                
                })
        }
    }
}
</script>

<style scoped>
ul {
  list-style-type: none;
  padding:10px;
  background-color:black;
  height:70vh;
  overflow:auto;
}
.yellows{
    color:rgb(39, 131, 235);
    text-align: right;
}
.greens{
    color:rgb(4, 172, 4);
    text-align: left;
}
.neutral{
    color:rgb(219, 243, 219);
}
.listItem{
    background-color: black;
}
</style>
