<script>
   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { createEventDispatcher } from 'svelte';
   import { 
      Modal, ModalHeader, ModalBody, ModalFooter, Button, Icon 
   } from 'sveltestrap';

   const dispatch = createEventDispatcher();

   export var active = false;                            // Indica janela vísivel ou não
   export var title = 'Confirm action';                  // String com o título a ser exibido
   export var message = 'Some shit is about to happen';  // String com a descrição do aviso

   /**
    * Tratamento do evento 'click' do botão de confirmação.
    * Despacha o evento 'confirm'.
    */
   function confirm() {
      active = false;
      dispatch('confirm');
   }

   /**
    * Tratamento do evento 'click' do botão de cancelamento.
    * Despacha o evento 'cancel'.
    */
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
