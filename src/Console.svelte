<script lang="ts">
   let TheConsole = null;                    // Referência ao objeto DOM correspondente ao console

   /**
    * Inclui uma ou mais linhas ao console (delimitadas por \n).
    * Rola (scroll) a janela se necessário.
    * @param str Linha (string) a ser impresso.
    * @param alert Código de cor para a mensagem ou null para cor padrão.
    */
   export function print(str, alert=null) {
      let html = TheConsole.innerHTML;
      if(alert) html += `<b style="color: ${alert}">`;
      for(let i=0; i<str.length; i++) {
         let chr = str.charAt(i);
         let ascii = str.charCodeAt(i);
         if(ascii == 10) html += "<br>\n"
         if(ascii < 31) continue;
         html += chr;
      }
      if(alert) html += '</b>';
      TheConsole.innerHTML = html;
      TheConsole.scrollTop = TheConsole.scrollHeight;
   }

   /**
    * Limpa todo o console.
    */
   export function clear() {
      TheConsole.innerHTML = "";
   }
</script>

<div class="theConsole" bind:this={TheConsole}>
</div>

<style>
   .theConsole { 
      font-family: monospace;
      font-size: 10px;
      background-color: azure;
      height: 100%;
      min-height: 100%;
      width: 100%;
      overflow: auto;
      user-select: none;
      -webkit-user-select: none;
   }
</style>
