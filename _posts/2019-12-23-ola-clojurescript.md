---
layout: post
title:  "Olá Clojurescript!"
date:   2019-12-23
categories: []
tags: []
---

Ao final desse tutorial você deve ter um projeto [clojurescript](https://clojurescript.org/)

Vou supor que o leitor:

- Que é entendido como utilizar a linha de comando de um sistema operacional tipo-unix

- Que há [java](https://adoptopenjdk.net/) e [node](https://nodejs.org/) instalados em sua maquina


## Mapa geral

```
DICA: Quando digo que algo deve ser "commitado", entenda que é algo importante que deve ser salvo.
Quando digo "não commitado", entenda que vc pode deletar aquele arquivo e ele será baixado/gerado sozinho novamente.
```

O projeto vai se chamar `ola-mundo`.

Vamos usar o [npm](https://nodejs.org/) para rodar o [shadow-cljs](https://shadow-cljs.github.io)

O [npm](https://nodejs.org/) é configurado via arquivo `package.json`

O `npm install` gera o arquivo `package-lock.json` que deve ser commitado e a pasta `node_modules`, que não deve ser commitada.

[shadow-cljs](https://shadow-cljs.github.io) é responsavel por "compilar" seu [clojurescript](https://clojurescript.org)

O [shadow-cljs](https://shadow-cljs.github.io) vai gerar milhares de arquivos na pasta `target`. Esses arquivos podem
ser deletados e não devem ser commitados

Para configurar o `shadow-cljs` precisamos de dois arquivos:

- O `shadow-cljs.edn`, que contem informações sobre "o que vc quer que eu compile"
- O `deps.edn` que declara as dependencias do projeto (bibliotecas que eu quero usar)

O shadow-cljs também aproveita dependencias diretamente do `package.json`

Apesar do shadow-cljs rodar via npm (node), ele precisará chamar o compilador de clojurescript, que precisa do java.

## Conferindo os requisitos
1. Vamos conferir se há `node`, `npm` e `java` instalados. Para isso digite os seguintes comandos no terminal.
{% highlight bash %}
$ node --version
v13.4.0
$ npm --version
6.12.1
$ java -version
openjdk version "1.8.0_232"
OpenJDK Runtime Environment (build 1.8.0_232-b09)
OpenJDK 64-Bit Server VM (build 25.232-b09, mixed mode)
{% endhighlight %}
## Criando projeto
1. Crie uma pasta chamada `ola-mundo`.

1. Crie o arquivo `deps.edn` dentro da pasta `ola-mundo`. Aqui vamos especificar onde nosso código está, e quais
dependencias pretendemos usar.
{% highlight clojure %}
{:paths ["src" "resources"]
 :deps  {org.clojure/clojure  {:mvn/version "1.10.1"}
         reagent/reagent      {:mvn/version "0.8.1"}
         thheller/shadow-cljs {:mvn/version "2.8.83"}}}
{% endhighlight %}

1. Crie o arquivo `shadow-cljs.edn` ao lado do `deps.edn`.  Neste arquivo configuramos o shadow-cljs para compilar o 
clojurescript, subir um servidor HTTP e servir o HTML que vamos criar.
{% highlight clojure %}
{:deps     true
 :dev-http {8080 ["target/public" "classpath:public"]}
 :builds   {:ola-mundo {:target     :browser
                        :output-dir "target/public"
                        :asset-path ""
                        :modules    {:main {:init-fn ola-mundo.client/main}}
                        :devtools   {:after-load ola-mundo.client/after-load}}}}
{% endhighlight %}
1. Crie o arquivo `package.json`. Neste arquivo configuramos o `npm` para baixar as dependencias que vamos usar 
{% highlight json %}
{
  "scripts": {
    "start": "shadow-cljs watch ola-mundo"
  },
  "dependencies": {
    "create-react-class": "15.6.3",
    "react": "16.12.0",
    "react-dom": "16.12.0",
    "shadow-cljs": "2.8.83"
  }
}
{% endhighlight %}
1. Crie as patas necessárias para criar o arquivo `resources/public/index.html`. Nele temos um HTML minimo para uma 
aplicação tipo-react
{% highlight html %}
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Olá mundo!</title>
</head>
<body>
<div id="app"></div>
<script src="/ola-mundo/main.js"></script>
</body>
</html>
{% endhighlight %}

1. Crie o arquivo `src/ola_mundo/client.cljs`. Repare que o `ola_mundo` no nome da pasta está com `_` enqunto em todos
outros lugares está com `-`. Este é o modo correto. Caso tenha curiosidade no motivo disso, procure sobre `munge`.

{% highlight clojure %}
(ns ola-mundo.client
  (:require [reagent.core :as r]))

(defn main
  []
  (prn :ok)
  (r/render
    [:div "Olá mundo!"]
    (.getElementById js/document "app")))

(defn after-load
  []
  (main))
{% endhighlight %}

1. Após criar estes rquivos, vc deve ter exatamente essa estrutura de diretórios:

```
.
├── deps.edn
├── package.json
├── package-lock.json
├── resources
│   └── public
│       └── index.html
├── shadow-cljs.edn
└── src
    └── ola_mundo
        └── client.cljs
```


## Iniciando o shadow-cljs

Execute `npm install && npm start`. Esse processo deve demorar MUITO na primeira vez (5 min talvez). Da segunda vez em diante
(incluindo novos projetos) esse tempo já deve cair para 1min. 

{% highlight bash %}
$ npm start
{% endhighlight %}


Uma vez o shadow-cljs rodando, conecte em [localhost:8080](http://localhost:8080). Daí para frente vc já pode editar
o arquivo `src/ola_mundo/client.cljs` e ver o resultado das alterções enquanto edita.
