<script lang="ts">
   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { onMount } from 'svelte';
   import { AceEditor } from "svelte-ace";
   import "brace/mode/verilog";
   import "brace/mode/vhdl";
   import "brace/theme/chrome";
   import "brace/ext/language_tools";
   import "brace/ext/searchbox";
   import { Split } from "@geoffcox/svelte-splitter";
   import EditToolbar from './EditToolbar.svelte';
   import Console from './Console.svelte';
   const fs = require('fs');
   const path = require('path');
   const os = require('os');
   const process = require('process');
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
   var platform = "";
   var exepath = "";

   let last_cwd = ".";
   let modo_vhdl = false;                    // true = VHDL, false = Verilog
   let lineerror = -1;

   /*
    * Mudar o título da janela ao carregar um novo arquivo.
    */
   $: changeTitle(arquivo);
   function changeTitle(nome) {
      let win = nw.Window.get();
      if(!nome) win.window.document.title = 'WAX IDE';
      else win.window.document.title = `WAX IDE  [${path.basename(nome)}]`;
   }

   /*
    * Função executada no início da aplicação.
    * Inclui tratamento do evento change ao objeto ace.editor.
    */
   function init() {
      platform = os.platform;
      exepath = nw.App.startPath;

      terminal.print('Welcome to ', 'black');
      terminal.print('WAX IDE', 'green');
      terminal.print(' v.');
      terminal.print('1.0.3', 'red');
      terminal.print(' - 2018-2021 Bruno Basseto (GPLv.3)\n');
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
      if(modo_vhdl) texto = `
library ieee;
use ieee.std_logic_1164.all;
use ieee.std_logic_arith.all;
use ieee.std_logic_unsigned.all;

entity top is
end entity;

architecture sim of top is 
begin
   process begin
      wait;
   end process;
end architecture;`;
      else texto = `
module top;
   initial begin
      $dumpfile("out.vcd");
      $dumpvars;
      $finish;
   end
endmodule`;
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

   function onMode(event) {
      if(event.detail) {
         editor.session.setMode('ace/mode/vhdl');
         modo_vhdl = true;
      } else {
         editor.session.setMode('ace/mode/verilog');
         modo_vhdl = false;
      }
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


   function set_workdir() {
      if(platform == 'win32') {
         last_cwd = process.cwd();
         process.chdir(exepath);
      }
   }

   function reset_workdir() {
      if(platform == 'win32') {
         process.chdir(last_cwd);
      }
   }

   function parse_error(line) {
      let n = line.match(/:(\d+):/);
      if(n && (n.length > 1) && (lineerror < 0)) lineerror = parseInt(n[1]);
      terminal.print(line);
   }

   /**
    * Executa um programa em um processo independente, transferindo sua saída para o objeto 'terminal'.
    * Hack para execução no Windows incluído (FIXME: talvez exista uma forma melhor de fazer isso).
    * @param {String} cmd Nome do arquivo a executar.
    * @param {Array} args Array com os parâmetros.
    * @param {function} done Função a ser chamada quando o programa encerrar.
    */
   function run_exe(cmd, args, done) {
      /*
       * Corrige o nome no caso do Windows.
       */
      if(platform == 'win32') cmd += '.exe';

     /*
      * Dispara um processo com spawn.
      */
     let proc = spawn(cmd, args);
     proc.stdout.on('data', (data) => terminal.print(data.toString()));
     proc.stderr.on('data', (data) => parse_error(data.toString()));
     proc.on('close', (res) => {
        if(done) done(res);
     });
   }

   /**
    * Procura por arquivos com a extensão vcd (Value Change Dump) no diretório atual.
    * @return {Array} Array com nomes de arquivo.
    */
   function find_vcd() {
      let files = fs.readdirSync('.');
      let res = files.filter(name => name.match(/\.vcd$/ig));
      return res;
   }

   /**
    * Limpa os arquivos temporários no diretório atual.
    */
   function do_cleanup() {
      lineerror = -1;
      if(fs.existsSync('out.sim')) {
         fs.unlinkSync('out.sim');
      }
      if(fs.existsSync('out.ghw')) {
         fs.unlinkSync('out.ghw');
      }
      let cfs = fs.readdirSync('.').filter(name => name.match(/\.cf$/ig));
      cfs.forEach(name => fs.unlinkSync(name));
      let vcd = find_vcd();
      vcd.forEach(name => fs.unlinkSync(name));
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
      * Inicia executável do compilador.
      */
     set_workdir();
     do_cleanup();
     //run_exe('iverilog', ['-o', 'out.sim', arquivo], (res) => {
     if(modo_vhdl) {
        run_exe('ghdl', ['-a', '--std=08', '-fsynopsys',  arquivo], (res) => {
           if(res == 0) {
              terminal.print('Compilation succesful\n', 'blue');
              if(run) do_run();
              else reset_workdir();
           } else {
              terminal.print('Compilation failed\n', 'red');
              if(lineerror > 0) editor.gotoLine(lineerror);
              reset_workdir();
           }
         });
      } else {
         run_exe('iverilog', ['-o', 'out.sim', arquivo], (res) => {
            if(res == 0) {
               terminal.print('Compilation succesful\n', 'blue');
               if(run) do_run();
               else reset_workdir();
            } else {
               terminal.print('Compilation failed\n', 'red');
               if(lineerror > 0) editor.gotoLine(lineerror);
               reset_workdir();
            }
         });
      }
   }

   /**
    * Executa o processo de simulação e inicia GTKWave ao final.
    */
   function do_run() {
//      run_exe('vvp', ['out.sim'], (res) => {
      if(modo_vhdl) {
         run_exe('ghdl', ['-e', '--std=08', '-fsynopsys', 'top'], (res) => {
            if(res == 0) {
               run_exe('ghdl', ['-r', '--std=08', '-fsynopsys', 'top', '--wave=out.ghw'], (res) => {
                  if(res == 0) {
                     terminal.print('Simulation succesful\n', 'blue');
                     do_wave();
                  } else {
                     terminal.print('Simulation failed\n', 'red');
                     if(lineerror > 0) editor.gotoLine(lineerror);
                     reset_workdir();
                  }
               });
            } else {
               terminal.print('Simulation failed\n', 'red');
               reset_workdir();
            }
         });
      } else {
         run_exe('vvp', ['out.sim'], (res) => {
            if(res == 0) {
               terminal.print('Simulation succesful\n', 'blue');
               do_wave();
            } else {
               terminal.print('Simulation failed\n', 'red');
               if(lineerror > 0) editor.gotoLine(lineerror);
               reset_workdir();
            }
         });
      }
   }

   /**
    * Executa o processo GTKWave.
    */
   function do_wave() {
      if(modo_vhdl) {
         run_exe('gtkwave', ['out.ghw'], () => reset_workdir());
      } else {
         let vcd = find_vcd();
         if(vcd.length == 0) {
            terminal.print('No .vcd files found\n', 'red');
            return;
         }
         run_exe('gtkwave', [vcd[0]], () => reset_workdir());
      }
   }
</script>

<div class="thePage">
   <EditToolbar bind:changed = {changed} 
                bind:fileName = {arquivo}
                on:load = {onFileLoad} 
                on:save = {onFileSave} 
                on:fnew = {onFileNew}
                on:search = {onSearch}
                on:mode = {onMode}
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
                           tabSize: 3,
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
