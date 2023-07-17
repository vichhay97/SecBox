<template>
 <v-container fluid class="ma-0 pa-0">
            <v-row align="center" class="ma-2">
              <TerminalInterface ref="infected_terminal" type="infected" color="#d917bf" :lines=this.infected_lines></TerminalInterface>
              <TerminalInterface ref="healthy_terminal" type="healthy" color="#13d3b6" :lines=this.clean_lines></TerminalInterface>
              <v-col v-if="this.combined_cli" cols="12" md="12" style="padding:0; padding-top: 8px">
                <v-text-field
                              v-if="this.combined_cli"
                              ref="CLI_text_field"
                              style="border:1px solid white;font-family:'Courier New'"
                              class="bg-black rounded-0 "
                              hint="Enter a command to interact with the sandboxes"
                              label="Enter Cmd"
                              single-line
                              v-model="cli_text"
                              v-on:keyup.enter="onEnter"
                              v-on:keyup.up="get_last_message('combined')"
                ></v-text-field>
                <p v-if="!this.can_send_cmd_healthy || !this.can_send_cmd_infected" style="color:red">Please wait for the response</p>
            </v-col>
            <v-col v-if="!this.combined_cli" class="pa-1 " cols="12" md="6">
                <v-text-field
                    v-if="!this.combined_cli"
                ref="CLI_text_field_infected"
                style="border:1px solid white;font-family:'Courier New'"
                class="bg-black rounded-0 "
                hint="Send to infected sandbox"
                label="Enter Cmd"
                single-line
                v-model="cli_text_infected"
                v-on:keyup.enter="onEnter"
                    v-on:keyup.up="get_last_message('infected')"
          ></v-text-field>
            </v-col>
              <v-col v-if="!this.combined_cli" class="pa-1" cols="12" md="6">
                <v-text-field
                    v-if="!this.combined_cli"
                ref="CLI_text_field_clean"
                style="border:1px solid white;font-family:'Courier New'"
                class="bg-black rounded-0 "
                hint="Send to clean sandbox"
                label="Enter Cmd"
                single-line
                v-model="cli_text_clean"
                v-on:keyup.enter="onEnter"
                    v-on:keyup.up="get_last_message('healthy')"
          ></v-text-field>
            </v-col>
            </v-row>
          </v-container>
          <v-btn color="primary" class="ma-2" @click="downloadScript('combined')">Download Combined Script</v-btn>
          <v-btn color="primary" class="ma-2" @click="downloadScript('infected')">Download Infected Script</v-btn>
          <v-btn color="primary" class="ma-2" @click="downloadScript('clean')">Download Clean Script</v-btn>
          <v-btn color="primary" class="ma-2" @click="uploadScript('combined')">Upload Combined Script</v-btn>
          <v-btn color="primary" class="ma-2" @click="uploadScript('infected')">Upload Infected Script</v-btn>
          <v-btn color="primary" class="ma-2" @click="uploadScript('clean')">Upload Clean Script</v-btn>

</template>

<script>
import TerminalInterface from "@/components/TerminalInterface";
//import io from "socket.io-client";


export default {
  name: "LiveTerminal",
    components: {TerminalInterface},
  props:{
    current_id: String,
    socket: Object,
    combined_cli: Boolean
  },
  data: () =>({
    ref : this,
    cli_text:"",
    cli_text_infected:"",
    cli_text_clean:"",
    infected_lines:[],
    clean_lines:[],
    can_send_cmd_healthy:true,
    can_send_cmd_infected:true,
    last_messages:[],
    last_message_h:[],
    last_message_i:[],
    infected_commands:[],
    clean_commands:[],
    combined_commands: [],
    infectedScript: '',
    cleanScript: '',
    combinedScript: ''
      }),
  created() {
   let ref = this
    this.socket.on("terminalOutput", function (data){
      ref.add_feedback(JSON.parse(data))
    });
  },

  methods:{
    generateScript: function(scriptType) {
      let commands;
      let script = '';
      
      if (scriptType === 'combined') {
        commands = this.combined_commands;
        this.combinedScript = script;
        } else {
          commands = scriptType === 'infected' ? this.infected_commands : this.clean_commands;
          }
          
      // Determine the first command timestamp
      let firstCommandTimestamp = commands.length > 0 ? commands[0].timestamp : 0;
      for (let commandObj of commands) {
        let relativeTimestamp = commandObj.timestamp - firstCommandTimestamp;
        script += `${commandObj.command} # Relative Timestamp: ${relativeTimestamp} ms\n`;
        }
        
        if (scriptType === 'infected') {
          this.infectedScript = script;
          } 
        else if (scriptType === 'clean') {
          this.cleanScript = script;
          } 
        else if (scriptType === 'combined') {
          this.combinedScript = script;
          }
  },
    downloadScript: function(scriptType) {
    this.generateScript(scriptType);

    const scriptContent = scriptType === 'infected' ? this.infectedScript : (scriptType === 'clean' ? this.cleanScript : this.combinedScript);
    const scriptName = scriptType + 'Script.txt';
    
    let element = document.createElement('a');
    element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(scriptContent));
    element.setAttribute('download', scriptName);

    element.style.display = 'none';
    document.body.appendChild(element);

    element.click();

    document.body.removeChild(element);
  },
  uploadScript: function(scriptType) {
    const inputElement = document.createElement('input');
    inputElement.type = 'file';
    inputElement.accept = '.txt';
    inputElement.onchange = (event) => {
      const file = event.target.files[0];
      const reader = new FileReader();
      reader.onload = () => {
        const lines = reader.result.split('\n').filter(line => line.trim().length > 0);
        const commands = lines.map(line => {
          const [command, timestampComment] = line.split('#');
          const timestamp = Number(timestampComment.replace('Relative Timestamp:', '').replace('ms', '').trim());
          return { command: command.trim(), timestamp };
        });

        // Set settings according to script type
        this.can_send_cmd_healthy = scriptType === 'combined' || scriptType === 'clean';
        this.can_send_cmd_infected = scriptType === 'combined' || scriptType === 'infected';

        this.runScript(commands);
      };
      reader.readAsText(file);
    };
    inputElement.click();
  },
  runScript: async function(commands) {
    for (let i = 0; i < commands.length; i++) {
      const command = commands[i].command;
      const delay = i === 0 ? 0 : commands[i].timestamp - commands[i-1].timestamp;

      await new Promise(resolve => setTimeout(resolve, delay));

      // Depending on the settings, use the appropriate CLI text field and send the command
      if (this.combined_cli) {
        this.cli_text = command;
        this.onEnter();
      } else if (this.can_send_cmd_healthy) {
        this.cli_text_clean = command;
        this.onEnter();
      } else if (this.can_send_cmd_infected) {
        this.cli_text_infected = command;
        this.onEnter();
      }
    }
  },
  sendMessage: function(message){
    //sending to server
    console.log(message)
    this.socket.emit('my event', { data: message });
    },
  onEnter:function(){
    // Gather current Timestamp
    const now = Date.now()

    if (this.combined_cli && (this.can_send_cmd_healthy && this.can_send_cmd_infected)){
      this.can_send_cmd_healthy = false;
      this.can_send_cmd_infected = false;
      this.last_messages.push(this.cli_text)
      this.combined_commands.push({command: this.cli_text, timestamp: now});
      this.socket.emit('cli command', { "room":this.current_id, "healthy_cmd": this.cli_text, "infected_cmd": this.cli_text});
              
      this.$refs.CLI_text_field.reset("");
    }
    else if (!this.combined_cli){
      if(this.cli_text_clean && this.can_send_cmd_healthy){
        if (this.cli_text_clean !== "" && this.cli_text_clean !== " "){
          this.last_message_h.push(this.cli_text_clean)
          this.clean_lines.push(this.cli_text_clean);
          this.clean_commands.push({command: this.cli_text_clean, timestamp: now});
          this.socket.emit('cli command', { "room":this.current_id, "healthy_cmd": this.cli_text_clean, "infected_cmd": ""});
        }
        this.$refs.CLI_text_field_clean.reset("");
      }
      if(this.cli_text_infected && this.can_send_cmd_infected){
        if (this.cli_text_infected !== "" && this.cli_text_infected !== " "){
          this.last_message_i.push(this.cli_text_infected)
          this.infected_lines.push(this.cli_text_infected)
          this.infected_commands.push({command: this.cli_text_infected, timestamp: now})
          this.socket.emit('cli command', { "room":this.current_id, "healthy_cmd": "", "infected_cmd": this.cli_text_infected});
                  
          this.$refs.CLI_text_field_infected.reset("");
        }
      }
    }
  },
  add_feedback(data){
    if (data["infectedStatus"] === "infected") {
      this.infected_lines.push(data["cmdOut"]);
      if (data["isLast"]){
        this.can_send_cmd_infected = true;
      }
      //time out to let everything render
      setTimeout(() => {this.$refs.infected_terminal.scrollToElement();}, 10);
      }
      else if (data["infectedStatus"] === "healthy") {
        this.clean_lines.push(data["cmdOut"]);
        if (data["isLast"]){
          this.can_send_cmd_healthy = true;
        }
        setTimeout(() => {this.$refs.healthy_terminal.scrollToElement();}, 10);
      }
  },
  get_last_message(qualifier){
    console.log(qualifier)
    console.log(this.last_messages)
    if (qualifier === "combined" && this.last_messages.length > 0){
      this.cli_text = this.last_messages[this.last_messages.length-1]
      console.log(this.cli_text)
    }
    else if(qualifier === "healthy" && this.last_message_h.length > 0){
      this.cli_text_clean = this.last_messages[this.last_messages.length-1]
    }
    else if(qualifier === "infected" && this.last_message_i.length > 0){
      this.cli_text_infected = this.last_message_i[this.last_message_i.length-1]
    }
  }
    },
  beforeUnmount: function() {
    this.socket.emit("leave room", {"room": this.current_id})
  }
}
</script>

<style scoped>

</style>