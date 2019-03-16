---
layout: post
title:  "Graphql API com clojure"
date:   2019-03-08
categories: []
tags: []
---

Esse post é uma edição comemorativa da criação da comunidade `graphql-brasil`, que já conta com

- [slack](https://graphql-brasil.slack.com) ([convites](https://graphql-slack.now.sh/))

- [github](https://github.com/graphql-brasil)

O fonte final está disponivel em [github.com/souenzzo/clj-graphql-tutorial](https://github.com/souenzzo/clj-graphql-tutorial)

## Criando projeto

1. Confira se há uma versão recente do clojure instalado na sua maquina

{% highlight bash %}
$ clj -Sdescribe
{:version "1.10.0.408"
 :config-files ["/usr/share/clojure/deps.edn" "/home/souenzzo/.clojure/deps.edn" ]
 :install-dir "/usr/share/clojure"
 :config-dir "/home/souenzzo/.clojure"
 :cache-dir "/home/souenzzo/.clojure/.cpcache"
 :force false
 :repro false
 :resolve-aliases ""
 :classpath-aliases ""
 :jvm-aliases ""
 :main-aliases ""
 :all-aliases ""}
{% endhighlight %}

1. Em um diretório vazio, crie um arquivo `deps.edn`

{% highlight clojure %}
{:paths   ["src" "resources"]
 :deps    {org.clojure/clojure              {:mvn/version "1.10.0"}
           com.walmartlabs/lacinia          {:mvn/version "0.32.0"}
           com.walmartlabs/lacinia-pedestal {:mvn/version "0.11.0"}
           io.pedestal/pedestal.service     {:mvn/version "0.5.5"}
           io.pedestal/pedestal.jetty       {:mvn/version "0.5.5"}
           org.clojure/java.jdbc            {:mvn/version "0.7.9"}
           org.postgresql/postgresql        {:mvn/version "42.2.5"}}
 :aliases {:dev {:extra-paths ["test"]
                 :extra-deps  {org.clojure/test.check {:mvn/version "0.10.0-alpha3"}}}}}
{% endhighlight %}

Neste arquivo, configuramos das dependencias do projeto.

No `path` colocamos `src`, onde ficará o códio e `resources`, onde ficarão nossos schemas

No `deps` temos:

* `org.clojure/clojure`
Clojure é apenas uma biblioteca. Podemos escolher sua versão aqui

* `com.walmartlabs/lacinia`
Implementação do graphql para clojure. Ela que nos da a função `(execute schema "{hello}" vars ctx) ;;=> {:data {:hello "world"}}`

* `com.walmartlabs/lacinia-pedestal`
Ajuda a adaptar o lacinia a funcionar via HTTP, usando o servidor HTTP `pedestal`. Também traz o graphiql

* `io.pedestal/pedestal.service`
Servidor HTTP pedestal. Usaremos algumas funcionalidades dele diretamente, principalmente para teste

* `io.pedestal/pedestal.jetty`
O pedestal permite vários backends. Usaremos o jetty, mas poderia ser immutant, tomcat ou mesmo aws lambda.

* `org.clojure/java.jdbc`
Adaptador para usar JDBC sem ter que instanciar 200 classes

* `org.postgresql/postgresql`
Backend usado no jdbc.

Crimos também em `aliases` um alias chamado `dev`, que adiciona a pasta de testes `test` e a dependencia de testes.

Neste momento, já podemos abrir o `REPL` para <strong>nunca mais fechar </strong>

{% highlight bash %}
$ clj -A:dev
Clojure 1.10.0
user=>
{% endhighlight %}

Depois de baixar algumas dependencias, deve aparecer um REPL para vc.

Agora vamos ao SQL.
Caso você não tenha um postgres rodando na sua porta 5432, execute:

{% highlight bash %}
$ docker run --name my-postgres --rm -p 5432:5432 postgres:alpine
{% endhighlight %}

Irá iniciar um postgres, que será destruido (incluindo dados) quando vc apetar `ctrl-c` no terminal.

## Interagindo com o SQL

Definiremos o schema do SQL oo arquivo `resources/schema.sql`.

{% highlight sql %}
CREATE TABLE usuario
(
  id   SERIAL NOT NULL UNIQUE PRIMARY KEY,
  nome TEXT   NOT NULL UNIQUE
);

CREATE TABLE todo
(
  id        SERIAL NOT NULL UNIQUE PRIMARY KEY,
  descricao TEXT
);

CREATE TABLE todos_do_usuario
(
  id      SERIAL NOT NULL UNIQUE PRIMARY KEY,
  usuario SERIAL NOT NULL REFERENCES usuario (id),
  todo    SERIAL NOT NULL UNIQUE REFERENCES todo (id)
);
{% endhighlight %}

De volta no seu REPL (você deixou aberto, né?), usaremos ele para instalar esse schema

Para isso, vamos usar a [jdbc/execute!](http://clojure.github.io/java.jdbc/#clojure.java.jdbc/execute!), que recebe 3 argumetnos:

1. um mapa com as configurações
1. Uma string com os comandos SQL
1. Um mapa de configurações extras.

Por padrão, o [jdbc/execute!](http://clojure.github.io/java.jdbc/#clojure.java.jdbc/execute!) engloba seu comando numa transação, por isso passamos o terceiro parametro desabilitando essa funcionalidade

No mapa de configuração é necessário dizer o `dbname`, que ainda não existe. Colocamos uma string vazia.

Depois de criar o database, aproveitamos para instalar o schema (dessa vez com o `dbname` já setado)

{% highlight clojure %}
user=> (require '[clojure.java.jdbc :as jdbc])
nil
user=> (jdbc/execute! {:dbtype "postgresql" :host "localhost" :user "postgres" :dbname ""} "CREATE DATABASE app;" {:transaction? false})
[0]
user=> (jdbc/execute! {:dbtype "postgresql" :host "localhost" :user "postgres" :dbname "app"} (slurp "resources/schema.sql"))
[0]
{% endhighlight %}


Essa tarefa que fizemos no REPL é util e provavelmente vamos querer ela algumas vezes. Por isso vou salvar ela num arquivo

`src/app/core.clj`
{% highlight clojure %}
(ns app.core
  (:require [clojure.java.jdbc :as jdbc]
            [clojure.java.io :as io])
(defn init-db!
  [db]
  (jdbc/execute! (assoc db :dbname "")
                 (str "DROP DATABASE IF EXISTS " (:dbname db) ";")
                 {:transaction? false})
  (jdbc/execute! (assoc db :dbname "")
                 (str "CREATE DATABASE " (:dbname db) ";")
                 {:transaction? false})
  (jdbc/execute! db (slurp (io/resource "schema.sql"))))

(def app-db
  {:dbtype "postgresql"
   :dbname "app"
   :host   "localhost"
   :user   "postgres"})

{% endhighlight %}

Alguns detalhes:

- Os arquivos clojure sempre devem declarar um namespace correspondente a sua posição

- Não é uma boa pratica usar namespaces não qualificados `(ns app ...)`

- Usamos `ìo/resource` para evitar a leitura de arquivos que não sejam recursos e para funcionar corretamente caso a
aplicação esteja rodando em um `.jar`

- Isso está longe de ser boas praticas de SQL ;)

De volta ao REPL, podemos brincar um pouco com o jdbc:

{% highlight clojure %}
user=> (require 'app.core)
user=> (in-ns 'app.core)
app.core=> (jdbc/query app-db ["SELECT * FROM usuario"])
[]
app.core=> (jdbc/insert! app-db :usuario {:nome "me"})
{:id 1 :nome "me"}
app.core=> (jdbc/query app-db ["SELECT * FROM usuario"])
[{:id 1 :nome "me"}]
{% endhighlight %}

## Interagindo com o GraphQL

Agora vamos ao graphql. Será que é possivel fazer um `{ hello }` em uma linha de lacinia?
{% highlight clojure %}
app.core=> (require '[com.walmartlabs.lacinia :as lacinia] '[com.walmartlabs.lacinia.schema :as schema])
app.core=> (lacinia/execute (schema/compile {:objects {:QueryRoot {:fields {:hello {:type 'String :resolve (constantly "mundo!")}}}}}) "{ hello }" {} {})
{:data #ordered/map ([:hello "mundo!"])}
{% endhighlight %}

Vamos entender:

- `lacinia/execute` recebe 4 argumentos: o schema compilado, a string de query, um mapa com as variaveis, e um mapa de contexto

- `schema/compile` gera um "schema compilado", baseado numa descrição de EDN

- No retorno, era esperado `{:data {:hello "mundo!"}}`, porém, como na especificação do GraphQL é dito que a ordem
dos pares `kv` no mapa importam e devem ser respeitadas, e o mapa padrão do clojure é indiferente quanto ordem, o lacinia
precisa usar esse tipo customizado, que se comporta como mapa 99% do tempo, porém na hora de serializar respeita a ordem.

Recomendo fortemente consultar os manuais e documentações do [lacinia](https://lacinia.readthedocs.io/en/latest/)

O lacinia permite que vc escreva seu schema usando o GraphQL SDL, e faremos isso

{% highlight clojure %}
app.core=> (require '[com.walmartlabs.lacinia.schema :as schema])
app.core=> (parser.schema/parse-schema "type QueryRoot { hello: String }" {:resolvers {:QueryRoot {:hello :look-at-me}}})
{:objects {:QueryRoot {:fields {:hello {:type String, :resolve :look-at-me}}}}}
{% endhighlight %}

Podemos observar que a `parse-schema` retorna uma estrutura de dados quase igual aquela que passamos como argumento
para o `schema/compile`, a menos do `:look-at-me`, que eu deveria ter posto uma função no lugar da keyword.

Você pode tentar fazer o `{hello}` novamente: `(-> (parse-schema "..." {...}) (compile) (execute ".." {} {}))`

Vamos jogar nosso schema em `resources/schema.graphql`

{% highlight graphql %}
type Todo {
    id: ID!
    descricao: String!
}
type Usuario {
    id: ID!
    nome: String!
    todos: [Todo!]
}
type MutationRoot {
    meCria(nome: String!): Usuario!
    novoTodo(descricao: String!, nome_usuario: String!): Usuario!
    deletaTodo(id: ID!): Usuario!
}
type QueryRoot {
    eu (nome: String!): Usuario!
}
{% endhighlight %}

Vale dizer que `QueryRoot` e `MutationRoot` é o nome padrão que o lacinia procura para seus roots. Outras implemntações
podem usar outros nomes.

Já podemos fazer uma função que pega esse schema e retorna o schema compilado do lacinia.

{% highlight clojure %}
(ns app.core
  (:require [com.walmartlabs.lacinia.schema :as schema]
            [com.walmartlabs.lacinia :as lacinia]
            [clojure.java.jdbc :as jdbc]
            [com.walmartlabs.lacinia.parser.schema :as parser.schema]
            [clojure.java.io :as io]))

(defn resolve-eu
  [_ args parent]
  (prn [:resolve-eu args parent])
  {:id 0 :nome "WIP"})

(defn me-cria
  [_ _ _]
  (comment
  ;; TODO
    []))

(defn novo-todo
  [_ _ _]
  (comment
  ;; TODO
    []))

(defn deleta-todo
  [_ _ _]
  (comment
  ;; TODO
    []))

(defn resolve-todos
  [_ args parent]
  (prn [:resolve-todos args parent])
  [])


(defn get-compiled-schema
  []
  (-> (io/resource "schema.graphql")
      (slurp)
      (parser.schema/parse-schema {:resolvers {:Usuario      {:todos resolve-todos}
                                               :QueryRoot    {:eu resolve-eu}
                                               :MutationRoot {:meCria     me-cria
                                                              :novoTodo   novo-todo
                                                              :deletaTodo deleta-todo}}})
      (schema/compile)))


(defn init-db!
  [db]
  (jdbc/execute! (assoc db :dbname "")
                 (str "DROP DATABASE IF EXISTS " (:dbname db) ";")
                 {:transaction? false})
  (jdbc/execute! (assoc db :dbname "")
                 (str "CREATE DATABASE " (:dbname db) ";")
                 {:transaction? false})
  (jdbc/execute! db (slurp (io/resource "schema.sql"))))

(def app-db
  {:dbtype "postgresql"
   :dbname "app"
   :host   "localhost"
   :user   "postgres"})

(def app-context
  {::jdbc/db app-db})

{% endhighlight %}

Os resolvers do lacinia: `resolve-todos`, `resolve-eu`, `me-cria`, `novo-todo` e `deleta-todo` recebem 3 argumentos:

1. O "contexto" da aplicação. Aquele que vem no ultimo argumento do `lacinia/execute`. Geralmente é um mapa, vc pode colocar
qualquer coisa lá.

2. Os argumentos da query/mutation. Se a query é `{ hello (id: 1) }`, no segundo argumento deve chegar `{:id 1}`

3. A entidade pai. Se temos a query `{ eu { id nome todos { id } } }` e `todos` tem um resolver proprio, então ele recebe o resutado
do resolve `eu` no terceiro argumento.

Coloquei alguns `prn` para tentar a ajudar a entender

Não fechou o REPL né? Ele deve ficar aberto para sempre!!!!!!

{% highlight clojure %}
app.core=> (require 'app.core :reload) ;; carregando as novidades do arquivo
app.core=> (lacinia/execute (get-compiled-schema) "{ eu(nome: \"mesmo\") { id nome todos { id } } }" {} app-context)
[:resolve-eu {:nome "mesmo"} nil] ;; 1
[:resolve-todos nil {:id 0, :nome "WIP"}] ;; 2
{:data #ordered/map ([:eu #ordered/map ([:id "0"] [:nome "WIP"] [:todos ()])])}
{% endhighlight %}

1. O prn da resolve-eu, mostrando que recebeu no primeiro parametro os argumentos passados em `eu` na query

2. O prn da resolve-todos, mostrndo que não há argumentos, porém á um "parent"

## API HTTP

Vamos aproveitar esse momento que temos uma query funcional e subir o servidor HTTP.

Faça o require de `[io.pedestal.http :as http]`, `[com.walmartlabs.lacinia.pedestal :as pedestal]` e adicione
no fim do `src/app/core.clj` o código:

{% highlight clojure%}
(defonce runnable-service (atom nil))

(defn run-dev
  "Para chamar no repl, desenvolvimento"
  [& args]
  (swap! runnable-service (fn [srv]
                            (when srv
                              (http/stop srv))
                            (-> #(get-compiled-schema)
                                (pedestal/service-map {:env         :dev
                                                       :app-context app-context
                                                       :graphiql    true})
                                #_(http/dev-interceptors)
                                (http/create-server)
                                (http/start)))))
{% endhighlight %}

Agora vc pode chamar no REPL o `(require app.core :reload)` e iniciar o servidor via `(run-dev)`.

Em http://localhost:8888 deve ter um playground `graphiql` onde vc pode tentar executar novamente nossa query `{ eu(nome: \"mesmo\") { id nome todos { id } } }`

## Implementando resolvers

Agora já pode tentar implementar essas funções.

Repare na variavel `app-context`, que é passada como "Contexto da aplicação" no execute.
Ela coloca o `db` em `::jdbc/db`. Para acessaar ela, precisamos pegar esse valor do contexto (primeiro argumento do resolver)

Exemplo de uma possivel implementação de "resolve-eu"

{% highlight clojure %}
(defn resolve-eu
  [{::jdbc/keys [db]} {:keys [nome]} _]
  (-> (jdbc/query db ["SELECT id,nome FROM usuario WHERE nome = ?"
                      nome])
      first))
{% endhighlight %}

A implementação está no [github](https://github.com/souenzzo/clj-graphql-tutorial)!

## Testes, testes, testes, testes

Na hora de implementar as funçoes, muitas vezes vc vai fazer reload, rodar novamente uma função e conferir o retorno.
Essa tarefa chata e repetitiva pode ser vacilitada com TESTES.

No github tem testes. Após copiar o arquivo de teste (e algumas funções que ele requer da app.core), basta rodar no REPL

{% highlight clojure %}
app.core=> (require '[clojure.test :refer [run-tests]])
app.core=> (run-tests 'app.core-test)
{% endhighlight %}
