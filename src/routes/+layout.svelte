<!--
 Copyright 2023 Google LLC

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
 -->

<script lang="ts">
  import '../app.css';
  import '../theme/theme.css';

  import '@material/web/button/elevated-button';
  import '@material/web/button/filled-button';
  import '@material/web/button/filled-tonal-button';
  import '@material/web/button/outlined-button';
  import '@material/web/button/text-button';
  import '@material/web/dialog/dialog';
  import '@material/web/divider/divider';
  import '@material/web/icon/icon';
  import '@material/web/iconbutton/filled-icon-button';
  import '@material/web/iconbutton/icon-button';
  import '@material/web/progress/circular-progress';
  import '@material/web/progress/linear-progress';
  import '@material/web/ripple/ripple';
  import '@material/web/slider/slider';
  import '@material/web/switch/switch';
  import '@material/web/textfield/filled-text-field';
  import '@material/web/textfield/outlined-text-field';

  // https://kit.svelte.dev/docs/configuration#version
  import { beforeNavigate } from '$app/navigation';
  import { updated } from '$app/stores';

  beforeNavigate(({ willUnload, to }) => {
    if ($updated && !willUnload && to?.url) {
      location.href = to.url.href;
    }
  });
</script>

<svelte:head>
  <title>Solarrechner – Meyer Group</title>
  <meta name="description" content="Entdecke das Solarpotenzial deines Dachs" />
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
  <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@600;700;800&family=Caveat:wght@500;600&display=swap" rel="stylesheet" />
  <link rel="preconnect" href="https://api.fontshare.com" />
  <link href="https://api.fontshare.com/v2/css?f[]=switzer@400,500,600,700&display=swap" rel="stylesheet" />
</svelte:head>

<main class="surface on-surface-text body-medium flex flex-col w-screen h-screen relative overflow-hidden">
  <slot />
</main>
