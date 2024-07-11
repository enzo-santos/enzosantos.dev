---
title: "O guia de 'dependências em Flutter' das galáxias"
date: 2024-05-04T19:00:00-03:00
draft: false
github_link: "https://github.com/enzo-santos"
author: "Enzo Santos"
tags:
  - Flutter
  - Dart
#image: /images/posts/organizando-lib-flutter.jpg
description: "Aprofundando sobre dependências em Flutter"
toc: 
---

Para auxiliar tanto quem está começando em Flutter e procura entender como
funciona as dependências nessa ferramenta, quanto quem já trabalha há
bastante tempo com a framework e sempre martela "flutter pub
upgrade", "flutter upgrade" e "flutter pub outdated" toda vez que tem
conflito de versões ao trabalhar em um projeto antigo, decidi montar este
guia bem resumido com algumas informações sobre gerenciamento de dependências
no Flutter, incluindo o pub.dev, os arquivos pubspec.yaml/.lock e as
categorias de dependências existentes.


## Introdução

O que são, afinal, *dependências*? Vamos sair do Flutter e defini-las no
contexto de software. Consideramos dependências como **todo código escrito
por alguém que reutilizamos em um código de um projeto**. Note que:

- um projeto pode conter zero dependências
- um projeto, caso seja reutilizável, pode servir como dependência de outro
  projeto futuro
- esse "alguém" pode ser nós mesmos; isto é, um código reutilizável de um
  outro projeto que queremos utilizar no projeto atual
- não é relevante a origem desse código reutilizável: se do Git, se
  do *pub.dev*, se do *npm*, se de um diretório local...


## pubspec.yaml

Por definição, **um pacote Dart é um diretório que possui nele um
arquivo *pubspec.yaml***. Como o Flutter nada mais é que um pacote Dart
especial, então o *pubspec.yaml* é obrigatório. Se o comando *flutter run*
for executado em um diretório vazio, o erro *"No pubspec.yaml file found"*
irá aparecer. Seu formato é o YAML, que é um tipo de mapeamento chave-valor
assim como o JSON. Algumas de suas chaves suportadas são `name` (o nome do
projeto), `description` (sua descrição), `version` (sua versão X.Y.Z),
`repository` (o link para seu código-fonte), e `dependencies`, que é a seção
que vamos ver com maiores detalhes neste guia.


## Listar vs baixar

Ao executar o comando

```shell
flutter create XYZ --no-pub
```

um projeto Flutter será criado do zero, mas sem o arquivo *pubspec.lock* que
estamos acostumados. Por quê? O argumento `--no-pub` impede que o comando
`flutter create` rode automaticamente o comando `flutter pub get` por debaixo
dos panos, que baixa as dependências listadas na seção `dependencies`
do *pubspec.yaml*. Note a diferença entre **listar** as dependências
(escrever seu nome e versão no *pubspec.yaml*) e **baixá-las**(rodar `flutter
pub get`). Como o intuito geralmente é baixar as dependências logo na criação
do projeto, isso é feita de forma automática pelo `flutter create`.

Por padrão, o Flutter baixa suas dependências do site
https://pub.dev/packages, repositório de pacotes do Dart. Suponha que você
queira uma dependência para realizar requisições HTTP. Ao pesquisar no
repositório de pacotes do Dart, você encontra o
https://pub.dev/packages/http, junto com a sua versão (atualmente 1.2.1).
Desta forma, você adiciona na seção `dependencies` do seu *pubspec.yaml* uma
chave `http`, com valor `^1.2.1` (note o acento circunflexo no início). Para
baixá-lo de fato para sua máquina local, você executa o comando `flutter pub
get`, que irá atualizar seu arquivo *pubspec.yaml* e baixar para sua máquina
todo o código dessa dependência.

```yaml
name: meuprojeto
description: "Meu projeto Flutter"
version: 1.0.0

dependencies:
  http: ^1.2.1
```

ou, alternativamente,

```shell
dart pub add http
# Baixa a versão mais recente de `http`
```


### Porquê o "^"?

O uso do circunflexo vai para além do Flutter, e está relacionado com
versionamento semântico. Para o *pub get*, declarar uma dependência com

- `abc: X.Y.Z` diz que você quer esta exata versão, sem mais nem menos. A
  desvantagem é que você ficará sem correções de bugs para esta dependência
- `abc:` diz que você quer a versão mais recente, não importa qual seja. O
  ruim é que você poderá quebrar seu código com uma versão
  backward-incompatible da biblioteca
- `abc: ^X.Y.Z` diz que você quer a versão mais recente, mas **garantindo que
  seu código não quebre por uma nova versão incompatível**


## pubspec.lock

O uso de arquivos lock também vai para além do Flutter: quem já viu o
*package-lock.json* ao rodar o comando `npm install`? Esse arquivo serve
basicamente para armazenar as versões exatas das dependências baixadas na
máquina local, tanto as que você listou no *pubspec.yaml* (diretas) quanto
as dependências dessas dependências (transitivas). Isso é útil quando outra
pessoa for clonar seu projeto, pois o comando `flutter pub get` agora irá
verificar primeiro o *pubspec.lock* e priorizar as dependências salvas lá,
evitando problemas de versão (mas apenas funciona caso a versão do Flutter
das duas máquinas seja igual).


## Tipos de dependências por escopo

Em paralelo às dependências diretas e transitivas, existem as dependências
principais (*main*) e de desenvolvimento (*dev*). Para entender melhor a
diferença, suponha que você queira criar uma biblioteca B que faça
requisições HTTP (com a biblioteca `http`) junto com testes unitários para
testar suas funções (com a biblioteca `mockito`). Considere os casos:

- Caso 1: você adiciona tanto a `http` quanto a `mockito` como
  dependências *main*
- Caso 2: você adiciona tanto a `http` quanto a `mockito` como
  dependências *dev*
- Caso 3: você adiciona a `http` como *main* e a `mockito` como *dev*

No caso 1, se você tem um projeto e adiciona B como dependência *main* em seu
*pubspec.yaml*, você irá baixar tanto `http` quanto `mockito` para sua máquina
local. Mas você não irá executar testes unitários na biblioteca B, apenas
utilizar suas funcionalidades de requisições HTTP, então não há necessidade
de você baixar a mockito junto. O caso 2 é ainda pior, pois não baixa nem a
`http` nem a `mockito`. O caso 3 é o ideal, pois você baixa apenas aquilo
que você vai usar no seu projeto. Então a regra é listar como *main* as
bibliotecas que você quer exportar, e como *dev* as que você quer utilizar de
forma interna, como testes e geração de modelos.

```yaml
name: meuprojeto
description: "Meu projeto Flutter"
version: 1.0.0

dependencies:
  http: ^1.2.1

dev_dependencies:
  mockito: ^5.4.4
```

ou, alternativamente,

```shell
dart pub add http
# Baixa a versão mais recente de `http`,
# adicionando às dependências principais por padrão

dart pub add dev:mockito
# Baixa a versão mais recente de `mockito`,
# adicionando às dependências de desenvolvimento
```


## Provedores de dependências

O comando

```shell
dart pub add --help
```

descreve seis formas de listar dependências no *pubspec.yaml*:

1. diretamente do pub.dev
2. de um diretório da máquina local
3. de um SDK (como o Flutter)
4. de um repositório Git
5. de um repositório Git, dado uma subpasta e um commit ou branch específico
6. de um domínio correspondente ao *pub.dev*, como uma versão autohospedada

Desta forma, você pode fazer o *fork* ou o *clone* de uma biblioteca que você
deseja alterar, realizar as alterações e utilizar as opções 2, 4 ou 5 para
receber essas mudanças mais recentes.


## Erros de versão

Quem nunca sofreu com o erro *"Because every version of X depends on Y ^A.B.C
and P depends on Y ^D.E.F, X is forbidden. So, because P depends on X,
version solving failed"*? Isso quer dizer que a dependência X está listando
uma versão antiga da dependência Y, e o seu projeto P está listando tanto X
quanto uma versão mais recente de Y. Basta ou

1. a dependência X atualizar seu *pubspec.yaml* para listar uma versão mais
recente de Y; ou 
2. seu projeto P listar uma versão mais antiga de Y.

Mas isso não é uma bala de prata: se não resolver, significa que você está no
temido *dependency hell*, ou "inferno de dependências" para os mais íntimos.

Uma das soluções para sair do *dependency hell* é pedir para o mantenedor de X
para atualizar as suas dependências, mas óbvio que isso se você tiver tempo
de sobra para esperar. Caso contrário, basta criar um *fork* de X, atualizar o
*pubspec.yaml* desta biblioteca para as versões mais recentes, e apontar o
*pubspec.yaml* do seu projeto para este *fork*. Claro que isso é uma solução a
curto prazo, visto que se o repositório original inserir alguma
funcionalidade nova, é preciso atualizar seu *fork* ou fazer merges
frequentemente. É por isso que, se você tiver uma biblioteca, é importante
sempre **lançar novas versões com o *pubspec.yaml* atualizado**.

Outro erro famoso é o *"Because every version of flutter from sdk depends on Z
^A.B.C and Y depends on Z ^D.E.F, flutter from sdk is incompatible with Y.
And because X depends on Y, flutter from sdk is incompatible with X. So,
because P depends on both flutter from sdk and X, version solving failed."*.
Apesar de ser um erro bem confuso de se entender, a solução é bem simples:
basta atualizar o Flutter para a versão mais recente, executando o comando
`flutter upgrade`. Neste caso, o Flutter lista uma versão antiga de Z, e seu
projeto lista tanto o Flutter quanto X, que lista Y, que por sua vez lista
uma versão mais recente de Z, gerando o conflito.
