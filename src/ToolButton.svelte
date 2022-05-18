<script>
   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { createEventDispatcher } from 'svelte';
   import { Button, Tooltip, Icon } from 'sveltestrap';

   export var name = "bot";               // Nome do botão, usado para gerar o evento
   export var icon = "";                  // Nome do ícone em bootstrap-icons
   export var tip = "Um botão";           // Ajuda a ser exibida com o mouse.

   const dispatch = createEventDispatcher();

   /**
    * Tratamento do evento 'click' dos botões.
    * Despacha o evento como 'bClick'.
    */
   function click() {
      dispatch('bClick', name);
   }
</script>

<div class="theButton">
   <Button color="light" id={name} on:click={click}><Icon name={icon}/></Button>
   <Tooltip target={name} bottom>{tip}</Tooltip>
</div>

<style>
</style>
