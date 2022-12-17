<script>

   const fs = require('fs');
   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { createEventDispatcher } from 'svelte';
   import { ButtonToolbar, Button, Tooltip, Icon } from 'sveltestrap';
   import Warning from "./Warning.svelte";
   import Toolbutton from "./ToolButton.svelte";
   import Switch from "./Switch.svelte";

   export var fileName;                      // nome do arquivo aberto ou escolhido pelo usuário
   export var changed = false;               // flag indicando alteração realizada pelo editor
   export var modo = false;                  // flag indicando o modo (false = verilog, true = VHDL)
   var newFileName;
   var loading = false;                      // flag sinalizando abertura de arquivo.
   
   const dispatch = createEventDispatcher();

   var save_dlg = null;                      // Referência ao objeto DOM do diálogo de salvamento de arquivo
   var open_dlg = null;                      // Referência ao objeto DOM do diálogo de abertura de arquivo
   let showModificationsWarning = false;     // Abrir aviso de arquivo modificado.

   /*
    * Monitorar e gerar um evento 'search' quando searchText for modificado.
    */
   var searchText = "";
   $: dispatch('search', searchText);

   /**
    * Tratamento do evento de seleção do modo (VHDL/Verilog)
    * Despacha o evento 'mode'.
    */
   function change() {
      dispatch('mode', !modo);
   }

   /**
    * Tratamento do evento de seleção de arquivo no diálogo de salvamento.
    * Despacha o evento 'save'.
    */
   function save_select() {
      newFileName = save_dlg.value;
      save_dlg.value = "";
      fileName = newFileName;
      dispatch('save', fileName);
   }

   /**
    * Tratamento do evento de seleção de arquivo no diálogo de abertura.
    * Solicita confirmação caso existam modificações não salvas, despacha o evento 'load' caso contrário.
    */
   function open_select() {
      newFileName = open_dlg.value;
      open_dlg.value = "";
      loading = true;
      if(changed) showModificationsWarning = true;
      else {
         fileName = newFileName;
         dispatch('load', fileName);
      }
   }

   /**
    * Tratamento do evento de confirmação de carregamento (eventual sobreposição).
    * Despacha o evento 'load' ou 'new', conforme a operação.
    */
   function loadConfirm() {
      if(loading) {
         fileName = newFileName;
         dispatch('load', fileName);
      } else dispatch('fnew');
   }

   /**
    * Processa os eventos os botões da barra.
    */
   function click(ev) {
      switch(ev.detail) {
         case 'bot-open':
            /*
             * Botão abrir: abre a janela de seleção de arquivo.
             */
            loading = true;
            open_dlg.click();
            break;

         case 'bot-new':
            /*
             * Botão novo: verifica se o arquivo foi modificado.
             */
            loading = false;
            if(changed) showModificationsWarning = true;
            else {
               dispatch('fnew');
            }
            break;

         case 'bot-save':
            /*
             * Botão save: verifica se o arquivo já tem um nome.
             */
            if(!fileName) save_dlg.click();
            else {
               dispatch('save', fileName);
            }
            break;

         default:
            /*
             * Demais botões: despacha o evento.
             */
            dispatch('bClick', ev.detail);
            break;
      }
   }
</script>

<div>
   <input bind:this={save_dlg} on:change={save_select} class="filesDialog" type="file" nwsaveas />
   <input bind:this={open_dlg} on:change={open_select} class="filesDialog" type="file" />
   <Warning 
      bind:active={showModificationsWarning} 
      message="There are unsaved modifications that will be lost!" 
      on:confirm = {loadConfirm}
   />

   <div class="thebar">
      <ButtonToolbar>
         <Toolbutton name="bot-new" icon="file-plus" tip="New" on:bClick={click} />
         <Toolbutton name="bot-open" icon="archive" tip="Open" on:bClick={click} />
         <Toolbutton name="bot-save" icon="save" tip="Save" on:bClick={click} />
         <div class="sep"></div>
         <Toolbutton name="bot-undo" icon="arrow-counterclockwise" tip="Undo" on:bClick={click} />
         <Toolbutton name="bot-redo" icon="arrow-clockwise" tip="Redo" on:bClick={click} />
         <Toolbutton name="bot-paste" icon="clipboard" tip="Paste" on:bClick={click} />
         <Toolbutton name="bot-copy" icon="back" tip="Copy" on:bClick={click} />
         <Toolbutton name="bot-cut" icon="scissors" tip="Cut" on:bClick={click} />
         <div class="sep"></div>
         <Toolbutton name="bot-compile" icon="check-square" tip="Compile" on:bClick={click} />
         <Toolbutton name="bot-run" icon="play" tip="Run" on:bClick={click} />
      </ButtonToolbar>
      <div class="search">
         <ButtonToolbar>
            <div class="switch">
               <span>Verilog</span>
               <Switch bind:checked={modo} on:click={change}></Switch>
               <span>VHDL</span>
            </div>
            <input type="search" name="search" placeholder="Search..." bind:value={searchText} />
            <Toolbutton name="bot-find" icon="search" tip="Find" on:bClick={click} />
         </ButtonToolbar>
      </div>
   </div>
</div>

<style>
   .filesDialog {
      display: none;
   }
   .sep {
      border-left: 1px solid black;
      margin-right: 5px;
      margin-left: 5px;
   }
   .thebar {
      border-bottom: 1px solid lightgrey;
      display: flex;
      flex-direction: row;
      justify-content: space-between;
   }
   input {
      background-color: light;
      border-radius: 8%/50%;
      padding-left: 20px;
      padding-right: 20px;
   }
   .switch {
      display: flex;
      flex-direction: row;
      column-gap: 3px;
      align-items: center;
      margin-right: 15px;
   }
</style>
