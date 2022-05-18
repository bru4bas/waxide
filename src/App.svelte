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

   var tamanho = 300;                        // tamanho vertical da área do editor, alterado dinamicamente
   var arquivo = "";                         // nome do arquivo que está sendo editado
   var texto = "";                           // conteúdo do arquivo que está sendo editado
   var editor = null;                        // Referência ao objeto ace.editor da janela
                                                   // IMPORTANTE
                                                   // o código de svelte-ace deve ser alterado para o objeto AceEditor seja acessível
                                                   // export let editor = Ace.editor;
   var terminal = null;                      // Referência ao objeto Console da janela
   var changed = false;                      // flag indicativa de arquivo modificado pelo editor
   var loading = false;                      // flag de carga de arquivo (invalidando changed)
   var searchText = "";                      // 
   var newsearch = true;
   var clipboard = nw.Clipboard.get();       // Referência ao objeto nw.Clipboard do sistema

   /*
    * Mudar o título da janela ao carregar um novo arquivo.
    */
   $: changeTitle(arquivo);
   function changeTitle(nome) {
      console.log("AQUI");
      let win = nw.Window.get();
      if(!nome) win.title = 'WAX IDE';
      else win.title = `WAX IDE  [${path.basename(nome)}]`;
   }

   /*
    * Função executada no início da aplicação.
    * Inclui tratamento do evento change ao objeto ace.editor.
    */
   function init() {
      console.log('PLATFORM = ' + os.platform);
      console.log('EXE PATH = ' + nw.App.startPath);
      editor.on('change', () => {
         if(!loading) changed = true;
         loading = false;
      });
   }
   onMount(() => init());

   /**
    * Tratamento do evento load
    * Lê o arquivo para a variável texto.
    */
   function onFileLoad(event) {
      arquivo = event.detail;
      texto = fs.readFileSync(arquivo).toString();
      changed = false;
      loading = true;
   }

   /**
    * Tratamento do evento 'new'
    * Esvazia o texto e limpa o nome do arquivo.
    */
   function onFileNew(event) {
      arquivo = "";
      changed = false;
      loading = true;
      texto = "module myModule;\n\nendmodule";
   }

   /**
    * Tratamento do evento 'save'
    * Salva 'texto' em 'arquivo' e limpa o flag de arquivo alterado.
    */
   function onFileSave(event) {
      arquivo = event.detail;
      fs.writeFileSync(arquivo, texto);
      changed = false;
   }

   /**
    * Tratamento do evento 'find'
    * Aciona findAll no ace.editor
    */
   function onSearch(event) {
      searchText = event.detail;
      newsearch = true;
      editor.findAll(searchText);
   }

   /**
    * Tratamento dos demais eventos da barra de botões.
    * Transfere para os respectivos métodos de ace.editor.
    */
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
         case 'bot-find':
            if(newsearch) editor.find(searchText);
            else editor.findNext();
            newsearch = false;
            break;
         case 'bot-compile':
            do_compile();
            break;
         case 'bot-run':
            do_compile(true);
            break;
      }
   }

   /**
    * Tratamento do evento 'change' do objeto split.
    * Ajusta o tamanho do editor (hack).
    */
   function onSplitChange(event) {
      tamanho = event.detail.primarySize;
   }

   /**
    * Executa o processo de compilação.
    * @param run Caso seja true, inicia a simulação ao final da compilação.
    */
   function do_compile(run = false) {
     terminal.clear();

     /*
      * Verifica estado atual do arquivo no editor.
      */
     if(arquivo == "") {
        terminal.print("Error: save document before compiling\n", 'red');
        return;
     }
     if(changed) {
        fs.writeFileSync(arquivo, texto);
        changed = false;
     }

     /*
      * Inicia executável do compilador e captura seu stdout/stderr.
      * Configura tratamento do evento 'close' (processo terminado).
      */
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

   /**
    * Executa o processo de simulação e inicia GTKWave ao final.
    */
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

   /**
    * Executa o processo GTKWave.
    */
   function do_wave() {
     let proc = spawn('gtkwave', ['dump.vcd']);
   }
</script>

<div class="thePage">
   <EditToolbar bind:changed = {changed} 
                bind:fileName = {arquivo}
                on:load = {onFileLoad} 
                on:save = {onFileSave} 
                on:fnew = {onFileNew}
                on:search = {onSearch}
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
