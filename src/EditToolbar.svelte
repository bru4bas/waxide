<script>

   const fs = require('fs');
   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { createEventDispatcher } from 'svelte';
   import { ButtonToolbar, Button, Tooltip, Icon } from 'sveltestrap';
   import Warning from "./Warning.svelte";
   import Toolbutton from "./ToolButton.svelte";

   export var fileName;
   export var changed = false;
   var newFileName;
   var loading = false;
   
   const dispatch = createEventDispatcher();

   var save_dlg = null;
   var open_dlg = null;
   let showOverwriteWarning = false;
   let showModificationsWarning = false;

   function save_select() {
      newFileName = save_dlg.value;
      save_dlg.value = "";
      if(fs.existsSync(newFileName)) showOverwriteWarning = true;
      else {
         fileName = newFileName;
         dispatch('save', fileName);
      }
   }

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

   function loadConfirm() {
      if(loading) {
         fileName = newFileName;
         dispatch('load', fileName);
      } else dispatch('fnew');
   }

   function overwriteConfirm() {
      fileName = newFileName;
      dispatch('save', fileName);
   }

   function click(ev) {
      switch(ev.detail) {
         case 'bot-open':
            loading = true;
            open_dlg.click();
            break;

         case 'bot-new':
            loading = false;
            if(changed) showModificationsWarning = true;
            else {
               dispatch('fnew');
            }
            break;

         case 'bot-save':
            if(!fileName) save_dlg.click();
            else {
               dispatch('save', { file: fileName });
            }
            break;


         default:
            dispatch('bClick', ev.detail);
            break;
      }
   }

</script>

<div class="thebar">
   <input bind:this={save_dlg} on:change={save_select} class="filesDialog" type="file" nwsaveas />
   <input bind:this={open_dlg} on:change={open_select} class="filesDialog" type="file" />
   <Warning 
      bind:active={showOverwriteWarning} 
      message="Existing file will be overwritten!" 
      on:confirm = {overwriteConfirm}
   />
   <Warning 
      bind:active={showModificationsWarning} 
      message="There are unsaved modifications that will be lost!" 
      on:confirm = {loadConfirm}
   />

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
      <Toolbutton name="bot-find" icon="search" tip="Find" on:bClick={click} />
      <div class="sep"></div>
      <Toolbutton name="bot-compile" icon="check-square" tip="Compile" on:bClick={click} />
      <Toolbutton name="bot-run" icon="play" tip="Run" on:bClick={click} />
   </ButtonToolbar>
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
   }
</style>
