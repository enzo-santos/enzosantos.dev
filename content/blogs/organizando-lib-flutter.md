---
title: "Organizando a pasta lib/ do seu projeto Flutter"
date: 2024-06-18T08:00:00-03:00
draft: false
github_link: "https://github.com/enzo-santos"
author: "Enzo Santos"
tags:
  - Flutter
  - Dart
#image: /images/posts/organizando-lib-flutter.jpg
description: ""
toc: 
---

Ao começar na framework do Flutter, geralmente se coloca todo o código em um único arquivo só, ou todos os arquivos Dart direto na pasta *lib/* do projeto. Apesar de parecer fácil de manusear quando o projeto está pequeno e só você está trabalhando nele, isso acaba tornando o código difícil de entender e de manter no médio e longo prazo. Uma forma de se solucionar isso é separando os arquivos do projeto em subdiretórios, tornando mais fácil de navegar pelo código conforme a aplicação vai crescendo ou quando você decide pegar aquele projeto de 3 meses atrás para trabalhar nele novamente. Uma organização que funciona bem comigo é a seguinte:

### screens/

Contém as telas do sistema. Recomenda-se uma tela por arquivo, contendo uma classe que herda de `StatelessWidget` ou `StatefulWidget` com a implementação dessa tela (comumente contendo um `Scaffold`) e zero ou mais classes com subcomponentes dessa tela (como cabeçalho, corpo, rodapé ou conteúdo lateral). Se esta tela utilizar uma classe gerenciadora de estado específica (como `ChangeNotifier` ou `Cubit`), também se encaixa adicioná-la neste arquivo.

```dart
class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return SafeArea(
      child: Scaffold(
        // TODO Implementar conteúdo da  tela
        body: Placeholder(),
      ),
    );
  }
}
```

### services/

Contém os serviços externos do sistema. Recomenda-se um serviço por arquivo, contendo uma classe abstrata (com os métodos do serviço a ser acessado) e uma ou mais classes concretas (com as implementações do serviço). Um serviço de autenticação, por exemplo, pode ter uma classe abstrata `AuthManager` com as classes concretas `FirebaseAuthManager` e `Auth0AuthManager`:

```dart
abstract class AuthManager {
  Future<void> signIn(String email, String password);

  Future<void> signOut();
}

class FirebaseAuthManager implements AuthManager {
  // TODO Implementar métodos
}


class Auth0AuthManager implements AuthManager {
  // TODO Implementar métodos
}
``` 

### widgets/

Contém componentes reutilizáveis do sistema. Recomenda-se um widget por arquivo, de preferência que seja customizável. Um widget de botão, por exemplo, pode ser customizável com os parâmetros `label` (com o texto do botão), `color` (com a cor do botão) e `onPressed` (com a ação do botão):

```dart
class Button extends StatelessWidget {
  final String label;
  final Color color;
  final void Function() onPressed;

  const Button({
    super.key,
    required this.label,
    required this.color,
    required this.onPressed,
  });

  @override
  Widget build(BuildContext context) {
    // TODO Implementar conteúdo
    return const Placeholder();
  }
}
```

### utils/

Contém funções, classes e extensões auxiliares na base de código. O conteúdo dessa pasta varia de projeto para projeto, mas sugestões de agrupamento possíveis podem ser por categorias (functions.dart, classes.dart, extensions.dart) ou por escopos (statistics.dart, ordering.dart, strings.dart).

### models/

Contém os modelos da regra de negócio do sistema. Recomenda-se um modelo por arquivo, contendo classes constantes e serializáveis (com métodos fromJson e toJson):

```dart
class User {
  final String name;
  final String email;
  final DateTime birthDate;

  factory User.fromJson(Map data) {
    return User(
      name: data['nome'] as String,
      email: data['email'] as String,
      birthDate: DateTime.parse(data['data-nascimento'] as String),
    );
  }

  const User({
    required this.name,
    required this.email,
    required this.birthDate,
  });

  Map toJson() {
    return {
      'nome': name,
      'email': email,
      'data-nascimento': birthDate.toIso8601String(), 
    };
  }
}
```

### providers/ ou blocs/

Contém os gerenciadores de estados reutilizáveis do sistema. Recomenda-se um gerenciador por arquivo. Por consistência, também se recomenda trabalhar com apenas uma biblioteca para gerenciar o estado ao invés de duas ao mesmo tempo. Os gerenciadores desse diretório devem ser reutilizáveis ao longo da aplicação, ao contrário dos gerenciadores específicos por tela.

## Conclusão

Esses subdiretórios não são uma regra e não devem ser necessariamente seguidos à risca, são apenas sugestões e podem ser adaptados conforme o seu caso de uso. Como disse anteriormente, é uma organização que funciona bem comigo, e espero que funcione com vocês também 🙂
