<script lang="ts">
   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { onMount } from 'svelte';
   import { AceEditor } from "svelte-ace";
   import "brace/mode/verilog";
   import "brace/theme/chrome";
   import "brace/ext/language_tools";
   import "brace/ext/searchbox";
   import { Split } from "@geoffcox/svelte-splitter";
   import EditToolbar from './EditToolbar.svelte';
   import Console from './Console.svelte';
   const fs = require('fs');
   const path = require('path');
   const os = require('os');
   const { spawn } = require('child_process');

   var tamanho = 300;
   var arquivo = "";
   var texto = "";
   var editor = null;
   var terminal = null;
   var changed = false;
   var loading = false;
   var clipboard = nw.Clipboard.get();

   onMount(() => editor.on('change', () => {
      if(!loading) changed = true;
      loading = false;
   }));

   function onFileLoad(event) {
      arquivo = event.detail;
      texto = fs.readFileSync(arquivo).toString();
      changed = false;
      loading = true;
   }

   function onFileNew(event) {
      arquivo = "";
      changed = false;
      loading = true;
      texto = "module myModule\n\nendmodule";
   }

   function onFileSave(event) {
      arquivo = event.detail;
      fs.writeFileSync(arquivo, texto);
      changed = false;
   }

   function onBClick(event) {
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
            do_compile();
            break;

         case 'bot-run':
            do_compile(true);
            break;
      }
   }

   function onSplitChange(event) {
      tamanho = event.detail.primarySize;
   }

   function do_compile(run = false) {
     terminal.clear();
     if(arquivo == "") {
        terminal.print("Error: save document before compiling\n", 'red');
        return;
     }

     let proc = spawn('iverilog', ['-o', 'out.sim', arquivo]);
     proc.stdout.on('data', (data) => terminal.print(data.toString()));
     proc.stderr.on('data', (data) => terminal.print(data.toString()));
     proc.on('close', (code) => {
        if(code == 0) {
           terminal.print('Compilation succesful\n\n\n\n\n', 'blue');
           if(run) do_run();
        } else terminal.print('Compilation failed\n\n\n\n\n', 'red');
      });
   }

   function do_run() {
     let proc = spawn('vvp', ['out.sim']);
     proc.stdout.on('data', (data) => terminal.print(data.toString()));
     proc.stderr.on('data', (data) => terminal.print(data.toString()));
     proc.on('close', (code) => {
        if(code == 0) {
           terminal.print('Simulation succesful\n\n\n\n', 'blue');
           do_wave();
        }
        else terminal.print('Simulation failed\n\n\n\n', 'red');
      });
   }

   function do_wave() {
     let proc = spawn('gtkwave', ['dump.vcd']);
   }

</script>

<div class="thePage">
   <EditToolbar bind:changed = {changed} 
                on:load = {onFileLoad} 
                on:save = {onFileSave} 
                on:fnew = {onFileNew}
                on:bClick = {onBClick} />
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
      <div class="divTerminal" slot="secondary">
         <Console bind:this={terminal} />
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

   .divTerminal { 
      height: 100%;
      min-height: 100%;
      width: 100%;
   }
</style>
