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
TIP: Quando digo que algo deve ser "commitado", entenda que é algo importante que deve ser salvo.
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
1. Vamos conferir se há `node`, `npm` e `java` instalados:
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

1. Crie o arquivo `shadow-cljs.edn`
{% highlight clojure %}
{:deps     true
 :dev-http {8080 ["target/public" "classpath:public"]}
 :builds   {:ola-mundo {:target     :browser
                        :output-dir "target/public/ola-mundo"
                        :asset-path "/ola-mundo"
                        :modules    {:main {:init-fn ola-mundo.client/main}}
                        :devtools   {:after-load ola-mundo.client/after-load}}}}
{% endhighlight %}
1. Crie o arquivo `deps.edn`
{% highlight clojure %}
{:paths ["src" "resources"]
 :deps  {org.clojure/clojure  {:mvn/version "1.10.1"}
         reagent/reagent      {:mvn/version "0.8.1"}
         thheller/shadow-cljs {:mvn/version "2.8.83"}}}
{% endhighlight %}
1. Crie o arquivo `package.json`
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
1. HTML Basico para seu projeto
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

1. Crie o arquivo com o código

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

## Iniciando o shadow-cljs

{% highlight bash %}
$ npm start
{% endhighlight %}
