js-fullstack-the-rightway
=========================

The right way to do a Full Stack System with Javascript using several concepts which are implemented separetedely


##Offline API First
**Offline API First? Uai mas se tá louco Suissa?**


Rapaz não é porque são 2 da manhã, hora que comecei a escrever, agora já são 3 ehhehee; e eu bebi cerveja para caraleo que estou loco, **EU SOU LOUCO!**

**Ok todos sabemos que você é louco, mas por que Offline API First? Não seria Offiline First e API First?**

Sim, não seria, é. Mas de me adianta ter um sistema no Frontend que use Offline First, mas que o Backend não tenha suporte na API para fazer a sincronização, logo de nada vai ajudar.

**Certo mas qual sua ideia com isso?**

Bom a ideia é criar uma metodologia para explicar como integrar essas 2 pontas utilizando um padrão comum de comunicação para que possa ser implementado independente da linguagem ou framework usado.

**Uhnnn interssante isso, mas como você faria?**

Bom vamos começar pelo Frontend onde é aplicado o Offline First. O que eu vejo muita gente fazendo é o seguinte:

1. executa uma ação
2. testa se esta online
  1. se sim então envia para API
    2. se não salva no LocalStorage
    
Você consegue ver algum problema nessa abordagem?

Não?

Qual o nome da metodologia? **Offline First**

Então por que cargas d'agua você não está persistindo localmente antes?

Uhnnnn, mas como assim?

Bom basicamente é só seguir esse workflow:

1. executa uma ação
2. presiste localmente
3. no callback testa se está online
  1. se sim envia os dados para API
    2. se não gera um **OPLOG** com as modificações que serão enviadas para a API sincronizar

**De onde você tirou esse OPLOG tio Suissa?**

Ai que mora o segredo de todo esse conceito. Porque quando o sistema estiver OFF ele irá gerar um JSON de OPLOG que nada mais é que um log de operações e eu tirei essa ideia do MongoDb pois é a forma que ele faz replicação.

Porém o OPLOG do MongoDb segue a seguinte estrutura:

```
   {
      "ts" : Timestamp(1395663575, 1),
      "h" : NumberLong("-5872498803080442915"),
      "v" : 2,
      "op" : "i",
      "ns" : "webschool.ebooks",
      "o" : {
          "_id" : ObjectId("533022d70d7e2c31d4490d22"),
          "author" : "JRR Hartley",
          "title" : "Flyfishing"
          }
      }
```

- **ts** é o timestamp e guarda quando o momento que foi gerado esse OPLOG.

- **h** é o identificador único daquela operação, como se fosse uma chave primária do nosso log.

- **v** é a versão do oplog utilizada, até hoje só mudou uma vez e pode ser ignorada

- **op** é o atributo da operação executada, ali no caso **i** é de **INSERT**, **u** para **UPDATE** e **d** para **DELETE**

- **ns** é o namespace no formato database.collection

- **o** é o nosso objeto a ser alterado, nesse caso inserido. Para o update utiliza-se mais um campo.

- **o2** que será qual query foi utilizada para buscar o objeto, normalmente é com o _id

        {
          "op" : "u",
          "ns" : "webschool.ebooks",
          "o2" : {
              "_id" : ObjectId("533022d70d7e2c31d4490d22")
          },
          "o" : {
              "$inc" : {
                  "stock" : -1
              }
        }

Nesse caso o atributto **o** irá conter o objeto modificador, nesse exemplo setando o valor de outofprint para **true**.

Bah mas que legal tio Suissa, mas e aí o que tem a ver o cu com a calça?

##OpLog
Utilizando o OPLOG como base podemos criar um padrão mais universal, estou aberto a melhorias e por favor me questionem hehehhehe.

    {
      "timestamp" : Timestamp(1395663575, 1),
      "ns" : "webschool.ebooks",
      action: "insert",
      query: "",
      val_old: "",
      val_new: {
          "author" : "Suissa",
          "title" : "Be MEAN"
          }
    }
    
Algumas considerações, por que usei val_old e query vazios?

Bom basicamente porque é uma inserção, logo você não possui um objeto anterior para ter o val_old e também não vai precisar da query, obviamente qnd você gerar o OPLOG não precisa colocar os atributos que são vazios. Porém essa estrutura se mantém para qualquer ação.

Agora uma questão mais importante, qual dos 2 JSONS você consegue ler e entender sem nenhum tipo de auxílio? 

O OPLOG do MongoDB, **ao meu ver** fere uma regra do Object Calisthenics.

>  Don’t abbreviate

Ai você pode me falar:

**Mas bah Suissa isso é um JSON não é um código.**

Perfeito ai quando você for consumir a API do OPLOG e receber aquele primeiro JSON, se você não estiver familiarizado, irá pesquisar o que é cada campo, caso receba o segundo ele se auto explica. Com isso eliminamos uma problemática simples. Agora se você é preocupado com o tamanho do JSON abrevie essa porra toda e **FODA-SE**. Aqui só estou dando uma ideia ainda.

Exemplo do resto do CRUD no OPLOG:


###Update
    {
      "timestamp" : Timestamp(1395663575, 1),
      "ns" : "webschool.ebooks",
      action: "update",
      query: "_id" : ObjectId("533022d70d7e2c31d4490d22")",
      val_old: "author" : "Suissa",
      val_new: {
      	$set : {"author" : "Doug Crock"},
      }
    }

###Delete
    {
      "timestamp" : Timestamp(1395663575, 1),
      "ns" : "webschool.ebooks",
      action: "delete",
      query: "_id" : ObjectId("533022d70d7e2c31d4490d22")",
      val_old: "",
      val_new: ""
    }


Ai você deve estar pensando:

**Esse porra do Suissa só fala dessa MERDA de MongoDb, mas será que posso usar isso no MariaDB?** pq MySQL de cu é rola

Claro que pode usar ai nesse caso só dependerá da sua função implementar as ações a serem tomadas a partir desse padrão de dados.

**Até ai muito legal seu Suissa, mas onde fica o caraio do API FIRST?**

Que ótimo que me perguntou meu incauto leitor, ai que mora o esquema do bagulho.

**--'**

Então nada iria me adiantar de gerar esse OPLOG no Frontend e não ter a sincronização implementada na API, por isso chamei de Fullstack Offliine API first, que nada mais é que a união dos 2 conceitos com uma padronização na comunicação para que esse conceito seja agnóstico de linguagem.

**Uhnnn, ainda não entendi.**

Beleza vamos lá.

##API

Vamos iniciar pensando em como deveria ser nossa rota para receber esse OPLOG.

> POST /api/sync

Ou seja vou enviar um **POST** para a rota `/api/sync` e ela deve receber o JSON e repassar ao sistema que deve entregar para o Controller que no caso pode ter a implementação da atualização do banco já que ele trabalha diretamente com o Model. Utilizando o nosso padrão como deveria ficar nossa função de sincronização?

    var date_created = oplog.timestamp;
    var action = oplog.action;
    var ns = oplog.ns.split(".");
    var db = ns[0];
    var collection = ns[1];
    
    switch(action){
      case 'insert':
        var data = ns.val_new;
        DALService('MongoDb').insert(data, callback);
        // o qual irá chamar uma função com
        // Model.save(data, callback)
      break;
      case 'update':
        var query = ns.query;
        var mod = ns.val_new;
        DALService('MongoDb').update(query, mod, callback);
      break;
      case 'delete':
        var query = ns.query;
        var data = ns.val_new;
        DALService('MongoDb').remove(query, callback);
      break;
    }

*Poderíamos criar um mediator para escolher qual ação que ele vai tomar para que mais ações possam ser adicionadas sem precisar de um switch*

Agora perceba que no UPDATE como eu passo:

	 val_new: {
      	$set : {"author" : "Doug Crock"},
      }

No `val_new` eu já envio como é a modificação a ser atualizada para que serviço de sincronização não precise conhecer nenhum comando em específico pois estou enviando o obejto de modificação inteiro



Essa query em SQL ficaria:

> UPDATE author FROM webschool.ebooks where id = 1

Ai que tá você até pode fazer mas fica horrível porque essa SQL está sendo enviada pelo Frontend, a melhor forma de gerenciar isso seria criar a SQL a partir dos dados do JSON, exemplo:

	var date_created = oplog.timestamp;
    var action = oplog.action;
    var ns = oplog.ns.split(".");
    var db = ns[0];
    var collection = ns[1];

	var sql = 'UPDATE FROM ' + ns + ' WHERE ' + query 
    
Que nesse caso seu valor de query deve ser 

> id = 1

Agora se você realmente quisesse utilizar a API do MongoDB, que por sinal é linda, o esquema é mapear as ações do MongoDB em SQL. Por exemplo se  enviar esse OPLOG para decrementar um livro do estoque.


    {
      "timestamp" : Timestamp(1395663575, 1),
      "ns" : "webschool.ebooks",
      "action": "update",
      "query": {
          "id" : 1
      },
      "val_old": {
          "author" : "Suissa"
      },
      "val_new": {
          "author" : "Doug Crock",
    }
}

Você precisa pegar o val_new que é a modificação em si e criar uma estrutura de decisão, por exemplo:


	var date_created = oplog.timestamp;
    var action = oplog.action;
    var ns = oplog.ns.split(".");
    var db = ns[0];
    var collection = ns[1];
    
    
    switch(action){
      case '$set':
        //blablabla
      break;
      case '$unset':
        //blablabla
      break;
      case '$inc':
        var query = ns.query;
        var modifier_operator = Object.keys ( oplog.val_new );
        var field = Object.keys (oplog.val_new[modifier_operator]);
        var value = oplog.val_new[modifier_operator][field];
        var where_field = Object.keys(oplog.query);
        var where_value = oplog.query[where_field];
        if(value < 0){
         //decrementa
         operator = '-=';
        }
         operator = '+=';
    
        var sql = 'UPDATE ' ns + '.' + field + operator + value + ' WHERE ' + where_field + '=' + where_value;
      break;
    }
    
Na moral gastei mó tempo para escrever esse exemplo e nem se vai gerar a SQL certa, mas também **FODA-SE!** 
*era só um exemplo e são 4 horas da manhã, já fazem 2 horas que estou escrevendo e log logo a cerveja acaba hehehehe*.   

Ou seja, mesmo utilizando uma API que se baseia no NoSQL podemos mapear as ações para o SQL, tá certo que é trampo PRA CARALEO, mas ai se vira qm curte relacional LOL. 

Acho que com isso já da para ter uma noção de como implemenar neh?

Aliás arguardem palestrinha nova sobre isso ;)
