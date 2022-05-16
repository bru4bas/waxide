<script>
   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { createEventDispatcher } from 'svelte';
   import { 
      Modal, ModalHeader, ModalBody, ModalFooter, Button, Icon 
   } from 'sveltestrap';

   const dispatch = createEventDispatcher();

   export var active = false;
   export var title = 'Confirm action';
   export var message = 'Some shit is about to happen';

   function confirm() {
      active = false;
      dispatch('confirm');
   }

   function cancel() {
      active = false;
      dispatch('cancel');
   }
</script>

<div>
   <Modal isOpen="{active}" {cancel}>
      <ModalHeader {cancel}>{title}</ModalHeader>
      <ModalBody>{message}</ModalBody>
      <ModalFooter>
         <Button color="primary" on:click={confirm}>Confirm</Button>
         <Button color="secondary" on:click={cancel}>Cancel</Button>
      </ModalFooter>
   </Modal>
</div>

<style>
</style>
