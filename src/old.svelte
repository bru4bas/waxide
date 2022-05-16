<script lang="ts">
   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { AceEditor } from "svelte-ace";
   import "brace/mode/verilog";
   import "brace/theme/chrome";
   import "brace/ext/language_tools";
   import "brace/ext/searchbox";
   import { 
      Container, Col, Row, Button, 
      ButtonToolbar, Icon 
   } from 'sveltestrap';
   import { Split } from "@geoffcox/svelte-splitter";
   import EditToolbar from './EditToolbar.svelte';
   const fs = require('fs');
   const { spawn } = require('child_process');

//   const fs=null;

   var tamanho = 300;
   var arquivo = "";
   var texto = "";
   var editor = null;
   var divConsole = null;
   var clipboard = nw.Clipboard.get();

   function onFileLoad(event) {
      console.log('Carregar ' + event.detail);
      arquivo = event.detail;
      texto = fs.readFileSync(arquivo).toString();
   }

   function onFileSave(event) {
      console.log('Salvar ' + event.detail);
   }

   function onBClick(event) {
      console.log('Botão apertado -> ' + event.detail);
      let str = "";
      switch(event.detail) {
         case 'bot-undo':
            editor.undo();
            break;
         case 'bot-redo':
            editor.redo();
            break;
         case 'bot-cut':
            str = editor.getCopyText();
            if(str) {
               clipboard.set(str, 'text');
               editor.insert("");
            }
            break;
         case 'bot-copy':
            str = editor.getCopyText();
            if(str) clipboard.set(str, 'text');
            break;
         case 'bot-paste':
            str = clipboard.get('text');
            if(str) editor.insert(str);
            break;

         case 'bot-compile':
            compile();
            break;
      }
   }

   function onSplitChange(event) {
      tamanho = event.detail.primarySize;
   }

   function compile() {
     let proc = spawn('ls', ['-lh', '/usr']);
     proc.stdout.on('data', (data) => print(data.toString()));
     proc.stderr.on('data', (data) => print(data.toString()));
     proc.on('close', (code) => print(`Process returned ${code}\n`));
   }

   function print(str) {
      let html = divConsole.innerHTML;
      for(let i=0; i<str.length; i++) {
         let chr = str.charAt(i);
         let ascii = str.charCodeAt(i);
         if(ascii == 10) html += "<br>\n"
         if(ascii < 31) continue;
         html += chr;
      }
      divConsole.innerHTML = html;
      divConsole.scrollTop = divConsole.scrollHeight;
   }

   function clear() {
      divConsole.innerHTML = "";
   }

</script>

<div class="thePage">
   <EditToolbar on:load={onFileLoad} on:save={onFileSave} on:bClick={onBClick}/>
   <Split horizontal 
          initialPrimarySize = "70%" 
          minPrimarySize = "50%" 
          minSecondarySize = "15%"
          on:changed = {onSplitChange} >
      <div slot="primary">
         <AceEditor 
              bind:value = {texto}
              bind:editor = {editor}
              width = "100%"
              height = {tamanho}
              lang = "verilog" 
              theme = "chrome"
              options = {{ fontSize: 14,
                           behavioursEnabled: true,
                           useSoftTabs: true,
                           enableLiveAutocompletion: true }}
         />
      </div>
      <div bind:this={divConsole} class="consoleDiv" slot="secondary">
      </div>
   </Split>
</div>

<style>
   :global(body) {
      margin: 0;
      padding: 0;
   }

   .thePage {
      padding: 0;
      margin: 0;
      height: 100%;
      width: 100%;
      overflow: hidden;
   }
   .consoleDiv { 
      font-family: monospace;
      font-size: 10px;
      background-color: azure;
      height: 100%;
      min-height: 100%;
      width: 100%;
      overflow: auto;
      user-select: none;
      -webkit-user-select: none;
   }
</style>
