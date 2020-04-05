---
layout: post
title:  "Aplicativo simples"
date:   2019-12-23
published: false
categories: []
tags: []
---

Ao final desse tutorial você deve ter um aplicativo tipo-[todo](http://todomvc.com/)

Vou supor que o leitor:

- Tem um projeto em clojurescript, como descrito em [Olá clojurescript]({% post_url 2019-12-23-ola-clojurescript %})

## Iniciando o projeto

Inicie seu projeto, se vc seguiu o [Olá clojurescript]({% post_url 2019-12-23-ola-clojurescript %}), basta rodar
`npm start` 

Uma vez o shadow-cljs rodando, conecte em [localhost:8080](http://localhost:8080)

## Input básico

Vamos começar criando um estado. Chamamos de estado algo que pode mudar, caso algo ocorr. No caso, `todo-atual` vai mudar
sempre que o usuário digitar algo 

{% highlight clojure %}
(ns ola-mundo.client
  (:require [reagent.core :as r]))

(defonce todo-atual (r/atom ""))

(defn main
  [] 
  (r/render
    [:div "Olá mundo!"]
    (.getElementById js/document "app")))

(defn after-load
  []
  (main))
{% endhighlight %}


