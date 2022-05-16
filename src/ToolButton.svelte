<script>
   import "../node_modules/bootstrap/dist/css/bootstrap.min.css";
   import "../node_modules/bootstrap-icons/font/bootstrap-icons.css";
   import { createEventDispatcher } from 'svelte';
   import { Button, Tooltip, Icon } from 'sveltestrap';

   export var name = "bot";
   export var icon = "";
   export var tip = "Um botão";

   const dispatch = createEventDispatcher();

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
