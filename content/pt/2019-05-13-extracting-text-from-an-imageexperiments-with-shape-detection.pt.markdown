---
slug: extracting-text-from-an-imageexperiments-with-shape-detection
date: 2019-05-13T12:39:45.039Z
title: 'Extracting text from an image: Experiments with Shape Detection'
link: 'https://copy-image-text.glitch.me/'
tags: [links, shape detection, pwa, intent, lumpy]
---
Eu tive um tempo de inatividade após o Google IO e queria coçar uma coceira a longo prazo que tive. Eu só quero copiar o texto contido nas imagens do navegador. Isso é tudo. Eu acho que seria um recurso interessante para todos.

Não é fácil adicionar funcionalidade diretamente ao Chrome, mas sei que posso tirar proveito do sistema de intenções no Android e agora posso fazer isso com a Web (ou pelo menos o Chrome no Android).

Duas novas adições à plataforma da Web - o Nível de Destino de Compartilhamento 2 (ou como eu gostaria de chamá-lo de Compartilhamento de Arquivos) e o `TextDetector` na API de Detecção de Forma - [have allowed me to build a utility that I can Share images to and get the text held inside them](https://copy-image-text.glitch.me/) .

A implementação básica é relativamente direta, você cria um destino de compartilhamento e um manipulador no Service Worker e, depois de ter a imagem que o usuário compartilhou, você executa o `TextDetector` nele.

O `Share Target API` permite que seu aplicativo da Web faça parte do subsistema de compartilhamento nativo e, nesse caso, agora você pode se registrar para lidar com todos os tipos de `image/*` , declarando-o dentro do `Web App Manifest` seguinte maneira.

```javascript
"share_target": {
  "action": "/index.html",
  "method": "POST",
  "enctype": "multipart/form-data",
  "params": {
    "files": [
      {
        "name": "file",
        "accept": ["image/*"]
      }
    ]
  }
}
```

Quando o seu PWA estiver instalado, você o verá em todos os lugares em que compartilha imagens, da seguinte maneira:

<figure><img src="/images/2019-05-13-extracting-text-from-an-imageexperiments-with-shape-detection-0.jpeg"></figure>

A API `Share Target` trata o compartilhamento de arquivos como uma postagem de formulário. Quando o arquivo é compartilhado com o Web App, o trabalhador do serviço é ativado, o manipulador `fetch` é chamado com os dados do arquivo. Os dados agora estão dentro do Service Worker, mas eu preciso deles na janela atual para poder processá-los, o serviço sabe qual janela invocou a solicitação, para que você possa direcionar facilmente o cliente e enviar os dados.

```javascript
self.addEventListener('fetch', event => {
  if (event.request.method === 'POST') {
    event.respondWith(Response.redirect('/index.html'));
    event.waitUntil(async function () {
      const data = await event.request.formData();
      const client = await self.clients.get(event.resultingClientId || event.clientId);
      const file = data.get('file');
      client.postMessage({ file, action: 'load-image' });
    }());
    
    return;
  }
  ...
  ...
}

```

Depois que a imagem está na interface do usuário, eu a processo com a API de detecção de texto.

```javascript
navigator.serviceWorker.onmessage = (event) => {  
  const file = event.data.file;
  const imgEl = document.getElementById('img');
  const outputEl = document.getElementById('output');
  const objUrl = URL.createObjectURL(file);
  imgEl.src = objUrl;
  imgEl.onload = () => {
    const texts = await textDetector.detect(imgEl);
    texts.forEach(text => {
      const textEl = document.createElement('p');
      textEl.textContent = text.rawValue;
      outputEl.appendChild(textEl);
    });
  };
  ...
};
```

O maior problema é que o navegador não gira naturalmente a imagem (como você pode ver abaixo), e a API de detecção de forma precisa que o texto esteja na orientação correta de leitura.

<figure><img src="/images/2019-05-13-extracting-text-from-an-imageexperiments-with-shape-detection-1.jpeg"></figure>

Usei o [EXIF-Js library](https://github.com/exif-js/exif-js) bastante fácil de usar para detectar a rotação e, em seguida, fiz alguma manipulação básica da tela para reorientar a imagem.

```javascript
EXIF.getData(imgEl, async function() {
  // http://sylvana.net/jpegcrop/exif_orientation.html
  const orientation = EXIF.getTag(this, 'Orientation');
  const [width, height] = (orientation > 4) 
                  ? [ imgEl.naturalWidth, imgEl.naturalHeight ]
                  : [ imgEl.naturalHeight, imgEl.naturalWidth ];

  canvas.width = width;
  canvas.height = height;
  const context = canvas.getContext('2d');
  // We have to get the correct orientation for the image
  // See also https://stackoverflow.com/questions/20600800/js-client-side-exif-orientation-rotate-and-mirror-jpeg-images
  switch(orientation) {
    case 2: context.transform(-1, 0, 0, 1, width, 0); break;
    case 3: context.transform(-1, 0, 0, -1, width, height); break;
    case 4: context.transform(1, 0, 0, -1, 0, height); break;
    case 5: context.transform(0, 1, 1, 0, 0, 0); break;
    case 6: context.transform(0, 1, -1, 0, height, 0); break;
    case 7: context.transform(0, -1, -1, 0, height, width); break;
    case 8: context.transform(0, -1, 1, 0, 0, width); break;
  }
  context.drawImage(imgEl, 0, 0);
}
```

E, se você compartilhar uma imagem com o aplicativo, ele girará a imagem e analisará retornando a saída do texto encontrado.

<figure><img src="/images/2019-05-13-extracting-text-from-an-imageexperiments-with-shape-detection-2.jpeg"></figure>

Foi incrivelmente divertido criar esse pequeno experimento e foi imediatamente útil para mim. No entanto, destaca as [inconsistency of the web platform](/the-lumpy-web/) . Essas APIs não estão disponíveis em todos os navegadores, nem em todas as versões do Chrome. Isso significa que, enquanto escrevo este artigo, o Chrome OS não posso usar o aplicativo, mas ao mesmo tempo quando posso usá-lo ... OMG, tão legal.

