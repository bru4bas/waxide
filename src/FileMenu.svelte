<script>

   const fs = require('fs');

   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { createEventDispatcher } from 'svelte';
   import { 
      ButtonToolbar, ButtonDropdown, DropdownMenu, DropdownItem,
      DropdownToggle, Icon 
   } from 'sveltestrap';
   import Warning from "./Warning.svelte";

   export var fileName;
   export var changed = false;
   var newFileName;
   
   const dispatch = createEventDispatcher();

   var fdlg = null;
   var loading = true;
   let showOverwriteWarning = false;
   let showModificationsWarning = false;

   function select() {
      newFileName = fdlg.value;
      if(loading) {
         if(changed) showModificationsWarning = true;
         else {
            fileName = newFileName;
            dispatch('load', { file: fileName });
         }
      } else {
         if(fs.existsSync(newFileName)) showOverwriteWarning = true;
         else {
            fileName = newFileName;
            dispatch('save', { file: fileName });
         }
      }
   }

   function load() {
      loading = true;
      fdlg.click();
   }

   function saveas() {
      loading = false;
      fdlg.click();
   }

   function save() {
      if(!fileName) saveas();
      else {
         dispatch('save', { file: fileName });
      }
   }

   function loadConfirm() {
      fileName = newFileName;
      dispatch('load', { file: fileName });
   }

   function overwriteConfirm() {
      fileName = newFileName;
      dispatch('save', { file: fileName });
   }

</script>

<div>
   <input bind:this={fdlg} on:change={select} class="filesDialog" type="file" />
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
      <ButtonDropdown>
         <DropdownToggle color="light">File</DropdownToggle>
         <DropdownMenu>
            <DropdownItem on:click={load}>Load</DropdownItem>
            <DropdownItem on:click={save} disabled={fileName==''}><Icon name="save"/> Save</DropdownItem>
            <DropdownItem on:click={saveas}>Save as...</DropdownItem>
            <DropdownItem divider></DropdownItem>
            <DropdownItem>Exit</DropdownItem>
         </DropdownMenu>
      </ButtonDropdown>
   </ButtonToolbar>
</div>

<style>
   .filesDialog {
      display: none;
   }
</style>
