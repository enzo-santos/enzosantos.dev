---
title: "Uma sugestão de design pattern para armazenamento em nuvem em Dart"
date: 2024-10-06T08:00:00-03:00
draft: false
github_link: "https://github.com/enzo-santos"
author: "Enzo Santos"
tags:
  - Flutter
  - Dart
#image: /images/posts/arquitetura-cache-dart.jpg
description: ""
toc: 
---

Suponha que você precise fazer um aplicativo de diagnóstico dermatológico que
utiliza imagens de lesões cutâneas para avaliações rápidas por IA e também
para análises detalhadas por especialistas. Os seguintes requisitos lhe são
repassados:

- O paciente deve enviar uma foto para uma triagem rápida usando IA
- O paciente deve enviar uma ou mais fotos para uma análise mais detalhada por um
  dermatologista
- Após um mês, o paciente deve enviar uma foto para monitoramento do
  tratamento sugerido pelo dermatologista

Você consegue implementar o sistema em Flutter, mas alguns pacientes reclamam
estar demorando muito para enviar uma foto (possivelmente por conta de uma
conexão lenta de internet). Então lhe é sugerido salvar as imagens primeiro
localmente, e depois pedir para o usuário enviá-las manualmente quando houver
uma melhor conexão com a internet.

## O problema

Para Android, existem bibliotecas no Flutter que facilitam esse processo:
[`path_provider`](https://pub.dev/packages/path_provider) para encontrar o
diretório específico da aplicação no armazenamento do dispositivo,
[`workmanager`](https://pub.dev/packages/workmanager) para executar tarefas
em segundo plano, e a biblioteca embutida [`dart:io`](https://dart.dev/libraries/dart-io) para criar diretórios e ler arquivos do
sistema operacional.

No entanto, o Flutter Web [não possui](https://github.com/flutter/flutter/issues/33577) um suporte
aprimorado a processamento em plano de fundo, além de [não permitir](https://github.com/miguelpruivo/flutter_file_picker/issues/548)
acessar os diretórios reais do dispositivo, inclusive como descrito na [própria documentação](https://dart.dev/libraries/dart-io):

> Only non-web Flutter apps, command-line scripts, and servers can import and use `dart:io`, not web apps.

Portanto, é preciso encontrar uma maneira simplificada e centralizada de realizar este gerenciamento de arquivos.

## Uma solução

Me deparei com este mesmo problema em uma aplicação que desenvolvi há uns meses.

A primeira parte da solução é encontrar bibliotecas que não tenham restrição
de execução na *web*.

Para os dois casos mencionados na seção anterior, não há o que ser feito: é
uma limitação atual do Flutter e a solução é não executar código que inclua
elementos dessas bibliotecas na plataforma *web*. Isso não significa que você
tenha que removê-las: o Flutter permite adicionar bibliotecas não compatíveis
com a *web* (como por exemplo, `path_provider` e `workmanager`) em seu
projeto Flutter que suporta *web*, assim como permite importá-las. O que não
é permitido é acessar métodos ou funções com implementações específicas,
recebendo a exceção 
["Unsupported operation: _Namespace"](https://stackoverflow.com/q/54861467)
caso seja utilizado a `dart:io` ou a exceção 
["UnimplementedError"](https://stackoverflow.com/q/65027450) 
para as demais bibliotecas.

A biblioteca [`universal_io`](https://pub.dev/packages/universal_io) pode ser utilizada para substituir a biblioteca `dart:io`, incluindo a 
maior parte de seus elementos, mas de uma maneira multiplataforma e que não causa conflitos. Seu uso principal é a classe 
[`Platform`](https://pub.dev/documentation/universal_io/latest/universal_io/Platform-class.html), que permite identificar a plataforma atual de onde
a aplicação está rodando.

A biblioteca [`cross_file`](https://pub.dev/packages/cross_file) é mais especializada para gerenciamento de arquivos, inclusive utilizada pela
biblioteca  [`file_picker`](https://pub.dev/packages/file_picker) para seleção de arquivos. Seu uso principal é a classe
[`XFile`](https://pub.dev/documentation/cross_file/latest/cross_file/XFile-class.html), que abstrai arquivos na *web* para `Uint8List`s
e arquivos locais para `File`s.

A segunda parte da solução é mais complicada, pois envolve criar uma arquitetura escalável para criação de arquivos locais, upload desses arquivos e suas
futuras remoções.

Após gastar algumas horas pensando no assunto, cheguei na seguinte arquitetura, que apelidei de **SPC** (_statement_-_procedure_-_command_):


<p align="center">
  <img src="/images/posts/uml_spc_design_pattern.png" />
</p>

```dart
sealed class StorageStatement {}

sealed class StorageProcedure {}

class StorageCommand {
  final StorageStatement statement;
  final StorageProcedure procedure;

  const StorageCommand({
    required this.statement,
    required this.procedure,
  });
}

abstract class StorageManager {
  Future<void> execute(StorageStatement statement);

  Stream<StorageCommand> evaluatePendingCommands();

  Future<void> commit(StorageCommand action);
}
```

### Instruções

A classe `StorageStatement` representa uma instrução de *upload* a ser realizada no
sistema. Por exemplo, nos requisitos acima, existem três
instruções: *upload* de uma foto para triagem; *upload* de uma ou mais fotos
para diagnóstico; e *upload* de uma foto para monitoramento do tratamento.
Dessa forma, as classes para representar estas declarações podem ser
definidas como

```dart
import 'package:cross_file/cross_file.dart' show XFile;

sealed class StorageStatement {}

class UploadScreeningImageStatement implements StorageStatement {
  final String patientId;
  final XFile file;

  const UploadScreeningImageStatement({
    required this.patientId,
    required this.file,
  );
}

class UploadDiagnosticImagesStatement implements StorageStatement {
  final String patientId;
  final List<XFile> files;

  const UploadDiagnosticImagesStatement({
    required this.patientId,
    required this.files,
  );
}

class UploadTreatmentImageStatement implements StorageStatement {
  final String patientId;
  final int year;
  final int month;
  final XFile file;

  const UploadTreatmentImageStatement({
    required this.patientId,
    required this.year,
    required this.month,
    required this.file,
  });
}
``` 

Note que esta classe não implementa nenhuma operação de armazenamento, sendo apenas
conceitual.

### Procedimentos

A classe `StorageProcedure` representa um procedimento a ser realizado após
uma operação de *upload* ter sido concluída com sucesso. Por exemplo, depois
que um arquivo salvo localmente já estiver na nuvem, não há necessidade de
tê-lo armazenado no dispositivo, então uma classe para representar a remoção
deste arquivo pode ser definida como

```dart
class RemoveFileStorageProcedure implements StorageProcedure {
  final String path;

  const RemoveFileStorageProcedure({required this.path});
}
```

Note que uma instrução pode envolver um ou mais arquivos (como a instrução de
diagnóstico) que serão armazenados localmente por um diretório, por exemplo.
Neste caso, após os arquivos deste diretório forem enviados para a nuvem,
pode-se executar um procedimento para excluir o diretório e todos os seus
arquivos, que pode ser definido como

```dart
class RemoveDirectoryStorageProcedure implements StorageProcedure {
  final String path;

  const RemoveDirectoryStorageProcedure({required this.path});
}
```

Por fim, pode existir o caso de que não haja nenhuma ação pós-upload
(como nenhum arquivo a ser limpo, por exemplo). Este procedimento pode ser
definido como

```dart
class NoActionStorageProcedure implements StorageProcedure {
  const NoActionStorageProcedure();
}
```

Note que esta classe também não representa nenhuma operação de I/O, sendo
apenas conceitual.

### Comandos

Não é necessário implementar a classe `StorageCommand` por ser uma classe concreta. Essa classe
relaciona instruções com seus respectivos procedimentos. Por exemplo, os possíveis relacionamentos 
do sistema são:

- a instrução `UploadScreeningImageStatement` com o procedimento `RemoveFileStorageProcedure`
- a instrução `UploadDiagnosticImagesStatement` com o procedimento `RemoveDirectoryStorageProcedure`
- a instrução `UploadTreatmentImageStatement` com o procedimento `RemoveFileStorageProcedure`

### Gerenciador

A classe `StorageManager` é a interface a ser utilizada para o gerenciamento desses arquivos locais e remotos. 
Deve haver uma implementação dessa classe sem utilizar armazenamento local (feita para *web*) e outra implementação utilizando o armazenamento local (feito para *mobile* ou *desktop*). 

Desta forma, o código inicial para o `StorageManager` pode ser escrito da seguinte forma:

```dart
abstract class StorageManager { 
  const factory StorageManager() = _StorageManager;

  const factory StorageManager.buffered(
    StorageManager manager
  ) = _BufferedStorageManager;
}

class _StorageManager implements StorageManager { 
  const _StorageManager();

  // ...
}

class _BufferedStorageManager implements StorageManager { 
  final StorageManager manager;

  const _BufferedStorageManager(this.manager);

  // ...
}
```

Note que, para reutilização de código, foi usado o padrão de *design* 
[*decorator*](https://en.wikipedia.org/wiki/Decorator_pattern) na segunda classe para adicionar a
funcionalidade de armazenamento local a um `StorageManager` já existente. Dessa forma, é possível
adicionar ou não a funcionalidade com o simples trecho de código:

```dart
import 'package:universal_io/universal_io.dart' show Platform;

StorageManager get storageManager {
  StorageManager manager = const StorageManager();
  if (Platform.isAndroid) {
    return StorageManager.buffered(manager);
  }
  return manager;
}
``` 

Agora, podemos implementar os métodos da interface em cada uma das subclasses.

#### Persistência

O método `commit` recebe como parâmetro um `StorageCommand` a ser executado.
Ao implementar esse método, deve-se obrigatoriamente persistir o arquivo na
nuvem e, caso se aplique, executar o método de limpeza pós-upload, descritos
respectivamente pelo `StorageStatement` e pelo `StorageProcedure` contidos no
`StorageCommand` recebido.

Uma implementação **sem armazenamento local** para este método pode ser
simplesmente enviar o arquivo descrito pela instrução do comando para a
nuvem, utilizando por exemplo a biblioteca [`http`](https://pub.dev/packages/http):

```dart
import 'package:http/http.dart' as http;
import 'package:cross_file/cross_file.dart' show XFile;

class _StorageManager implements StorageManager {
  // ...

  Future<void> commit(StorageCommand command) async {
    switch (command.statement) {
      case UploadScreeningImageStatement(:String patientId, :XFile file):
        final String url = 'https://example.com/api/patients/$patientId/screening';
        final http.MultipartRequest request = http.MultipartRequest('POST', Uri.parse(url));
        request.files.add(http.MultipartFile.fromBytes('file', await file.readAsBytes()));
        await request.send();

      case UploadDiagnosticImagesStatement(:String patientId, :List<XFile> files):
        final String url = 'https://example.com/api/patients/$patientId/diagnostic';
        final http.MultipartRequest request = http.MultipartRequest('POST', Uri.parse(url));
        for (int i = 0; i < files.length; i++) {
          final XFile file = files[i];
          request.files.add(http.MultipartFile.fromBytes('file$i', await file.readAsBytes()));
        }
        await request.send();

      // TODO Implementar para a instrução de tratamento
    }
  }

  // ...
}
```

Como esta implementação do `StorageManager` não utiliza armazenamento local,
não é necessário realizar nenhuma ação pós-upload, simplesmente podendo
ignorar o campo `StorageProcedure` do objeto `command`.

Já para a implementação **com armazenamento local** para este método deve
levar em consideração a ação pós-upload, realizando uma ação com base no
valor da `StorageProcedure`:

```dart
import 'package:universal_io/io.dart';

class _BufferedStorageManager implements StorageManager {
  final StorageManager manager;

  // ...

  Future<void> commit(StorageCommand command) async {
    // Executa o procedimento original de `commit`
    await manager.commit(command);

    // Executa a ação pós-upload
    switch (command.procedure) {
      case RemoveFileStorageProcedure(:String path):
        await File(path).delete();
      case RemoveDirectoryStorageProcedure(:String path):
        await Directory(path).delete(recursive: true);
      case NoActionStorageProcedure():
        return;
    }
  }

  // ...
}
```

#### Execução

O método `execute` recebe como parâmetro um `StorageStatement` a ser
executado. Ao implementar esse método, deve-se considerar a ação a ser
realizada no sistema quando o usuário finaliza a inserção das fotos: deve ser
enviado para a nuvem diretamente ou deve ser armazenada localmente para envio
posterior?

Uma implementação **sem armazenamento local** considera o envio direto para a
nuvem, que basicamente é a mesma implementação do método `commit`:

```dart
class _StorageManager implements StorageManager {
  // ...

  Future<void> execute(StorageStatement statement) async {
    await commit(StorageCommand(
      statement: statement,
      procedure: const NoActionStorageProcedure(),
    ));
  }

  // ...
}
``` 

Já uma implementação **com armazenamento local** considera o salvamento local
sem enviar (por enquanto) nenhuma requisição para o servidor em nuvem:

```dart
import 'package:universal_io/io.dart';
import 'package:cross_file/cross_file.dart' show XFile;
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart' as pp;

class _BufferedStorageManager implements StorageManager {
  // ...

  Future<void> execute(StorageStatement statement) async {
    final Directory dir = await pp.getApplicationDocumentsDirectory();
    final String cacheDirPath = p.join(dir.path, 'cache');
    switch (statement) {
      case UploadScreeningImageStatement(:String patientId, :XFile file):
        final String path = p.join(cacheDirPath, 'screenings');
        await file.saveTo(p.join(path, patientId));

      case UploadDiagnosticImagesStatement(:String patientId, :List<XFile> files):
        final String path = p.join(cacheDirPath, 'diagnostics', patientId);
        for (XFile file in files) {
          await file.saveTo(p.join(path, p.basename(file.path)));
        }

      // TODO Implementar para a instrução de tratamento
    }
  }

  // ...
}
``` 


#### Busca

O método `evaluatePendingCommands` não recebe nada como parâmetro e deve
retornar quais `StorageStatement`s ainda não foram persistidos em nuvem ao
chamar o método `execute`.

Na implementação **sem armazenamento local**, na prática, todos os
`StorageStatement`s são persistidos ao chamar `execute`. Dessa forma, basta
retornar uma `Stream` vazia:

```dart
class _StorageManager implements StorageManager {
  // ...

  Stream<StorageCommand> evaluatePendingCommands() => const Stream.empty();

  // ...
}
```

Na implementação **com armazenamento local**, no entanto, é preciso verificar 
no armazenamento do dispositivo, na pasta definida na implementação do `execute`,
quais arquivos ainda estão presentes:

```dart
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart' as pp;

class _BufferedStorageManager implements StorageManager {
  // ...

  Stream<StorageCommand> evaluatePendingCommands() async* {
    final Directory dir = await pp.getApplicationDocumentsDirectory();

    // Encontra arquivos locais no diretório de triagem
    final Directory screeningsDir = Directory(p.join(dir.path, 'screenings'));
    await for (FileSystemEntity entity in screeningsDir.list()) {
      final FileStat stat = await entity.stat();
      // Filtra apenas arquivos no diretório
      if (stat.type != FileSystemEntityType.file) continue;
      final String path = entity.absolute.path;

      final StorageStatement statement = UploadScreeningImageStatement(
        // Busca o nome do paciente pelo nome do arquivo na pasta
        patientId: p.basename(path),
        // Cria um `XFile` referenciando o arquivo na pasta
        file: XFile(path),
      );
      // Informa que o arquivo local deve ser removido após o upload
      final StorageProcedure procedure = RemoveFileStorageProcedure(
        file: path,
      );
      yield StorageCommand(statement: statement, procedure: procedure);
    }

    // Encontra arquivos locais no diretório de diagnósticos
    final Directory diagnosticsDir = Directory(p.join(dir.path, 'diagnostics'));
    await for (FileSystemEntity entity in diagnosticsDir.list()) {
      final FileStat stat = await entity.stat();
      // Filtra apenas subpastas no diretório
      if (stat.type != FileSystemEntityType.directory) continue;
      final String path = entity.absolute.path;

      // Busca o nome do paciente pelo nome da subpasta
      final String patientId = p.basename(path);

      // Lista os conteúdos da subpasta
      final List<XFile> files = [];
      final Directory directory = Directory(path);
      await for (FileSystemEntity entity in diagnosticsDir.list()) {
        final FileStat stat = await entity.stat();
        // Filtra apenas arquivos na subpasta
        if (stat.type != FileSystemEntityType.file) continue;
        final String path = entity.absolute.path;

        // Cria um `XFile` referenciando o arquivo na subpasta
        final XFile file = XFile(path);
        files.add(file);
      }
      if (files.isEmpty) continue;

      final StorageStatement statement = UploadScreeningImageStatement(
        patientId: patiendId,
        files: files,
      );
      // Informa que a subpasta deve ser removida após o upload
      final StorageProcedure procedure = RemoveDirectoryStorageProcedure(
        file: path,
      );
      yield StorageCommand(statement: statement, procedure: procedure);
    }

    // TODO Implementar para a instrução de tratamento
  }

  // ...
}
```

## Na prática

Já que temos as nossas implementações do `StorageManager` criadas, como
utilizar essa interface na prática nas aplicações Dart/Flutter?

Suponha uma tela simples do Flutter para envio de arquivo da triagem com dois
botões: um botão A nomeado "fazer upload" para selecionar um arquivo local,
que chama o método
[`pickFiles`](https://pub.dev/documentation/file_picker/latest/file_picker/FilePicker/pickFiles.html)
da biblioteca `file_picker`; e um botão B nomeado "concluir envio" para
finalizar o envio do arquivo. 

O botão B deve ser implementado para acessar
uma instância de um `StorageManager` (utilizando um serviço como o
[`get_it`](https://pub.dev/documentation/file_picker/latest/file_picker/FilePicker/pickFiles.html)
ou uma implementação de um 
[*singleton*](https://en.wikipedia.org/wiki/Singleton_pattern)) e chamar o método
`execute`, passando como parâmetro uma instância do objeto
`UploadScreeningImageStatement`, com o ID do paciente e o arquivo obtido pelo
botão A.

Uma possível implementação seria:

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:file_picker/file_picker.dart';
import 'package:get_it/get_it.dart';

import '../services/storage.dart' show StorageManager;

class _ImageNotifier extends ValueNotifier<XFile?> {
  _ImageNotifier() : super(null);
}

class ScreeningFormScreen extends StatelessWidget {
  final String patientId;

  const ScreeningFormScreen({
    super.key,
    required this.patientId,
  });

  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        ChangeNotifierProvider<_ImageNotifier>(
          create: (_) => _ImageNotifier(),
        ),
      ],
      builder: (context, _) {
        return Scaffold(
          body: Column(
            children: [
              // Botão A
              TextButton(
                onPressed: () async {
                  final _ImageNotifier notifier = context.read();
                  final FilePickerResult? result = await FilePicker.instance.pickFiles();
                  if (result == null) return;
                  final List<XFile> files = result.xFiles;
                  if (files.isEmpty) return;
                  final XFile file = files[0];
                  notifier.value = file;
                },
                child: Text('fazer upload'),
              ),
              // Botão B
              TextButton(
                onPressed: () async {
                  final _ImageNotifier notifier = context.read();
                  final XFile? file = notifier.value;
                  if (file == null) return;

                  final StorageManager manager = GetIt.instance.get<StorageManager>();
                  await manager.execute(
                    UploadScreeningImageStatement(patientId: patientId, file: file),
                  );
                },
                child: Text('concluir envio'),
              ),
            ],
          ),
        );
      },
    );
  }
}
```


Agora suponha outra tela do Flutter para exibição dos arquivos da triagem já
enviados pelo paciente com dois elementos: uma imagem A, que exibe a foto enviada
pelo paciente (estando local ou em nuvem); e um botão B nomeado "sincronizar
agora", exibido apenas se a foto não estiver sincronizada com a nuvem.

A tela deve ter uma lógica para

1. obter uma instância de um `StorageManager`
2. listar seus `StorageCommand`s pendentes, pelo método `evaluatePendingCommands`
3. verificar se pelo menos um dos comandos retornados possui como `statement`
  um `UploadScreeningImageStatement`, e se seu `patientId` for igual ao ID
  do paciente cuja tela está sendo exibida
    1. caso haja um comando com esse predicado, significa que o arquivo ainda não
      foi sincronizado com nuvem: deve-se acessar o campo `file` do comando
      encontrado e exibir a imagem A com seu conteúdo (usando [`Image.memory`](https://api.flutter.dev/flutter/widgets/Image/Image.memory.html), por
      exemplo), além de exibir o botão B para permitir a sincronização manual com
      o usuário utilizando o método `commit` no `StorageManager` obtido no passo 1
    2. caso não haja um comando com esse predicado, significa que o arquivo já foi
      sincronizado com a nuvem: deve-se acessar a *endpoint* respectiva para
      obter a imagem dessa parte da aplicação e exibir a imagem A com seu
      conteúdo (usando [`Image.network`](https://api.flutter.dev/flutter/widgets/Image/Image.network.html),
      por exemplo), além de ocultar o botão B
4. após a sincronização manual, caso ocorra no passo 3.1, deve-se repetir o passo 2
   para atualizar o estado da sincronização na tela

Uma possível implementação seria:

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:get_it/get_it.dart';

// Objeto para armazenar o arquivo a ser salvo em nuvem e seu respectivo comando.
class _NotifierValue {
  final StorageCommand command;
  final XFile file;

  const _NotifierValue({
    required this.command,
    required this.file,
  });
}

class _Notifier extends ChangeNotifier {
  final String patientId;

  AsyncSnapshot<_NotifierValue?> _snapshot = const AsyncSnapshot.waiting();

  static Future<_NotifierValue?> _checkForPendingFiles() async {
    // Passo 1
    final StorageManager manager = GetIt.instance.get<StorageManager>();

    // Passo 2
    final Stream<StorageCommand> commands = manager.evaluatePendingCommands();

    // Passo 3
    await for (StorageCommand command in commands) {
      switch (command.statement) {
        case UploadScreeningImageStatement(:String patientId, :XFile file):
          if (this.patientId == patientId) {
            // Passo 3.1
            return _NotifierValue(command: command, file: file);
          }
        }
    }
    // Passo 3.2
    return null;
  }

  _Notifier({required this.patientId}) {
    _checkForPendingFiles().then((command) {
      _snapshot = AsyncSnapshot.withData(ConnectionState.done, command);
      notifyListeners();
    });
  }

  AsyncSnapshot<_NotifierValue?> get snapshot => _snapshot;

  void synchronize(_NotifierValue value) async {
    // Passo 3.1
    final StorageManager manager = await GetIt.instance.get<StorageManager>();
    await manager.commit(value.command);

    // Passo 4
    final _NotifierValue? value = await _checkForPendingFiles();
    _snapshot = AsyncSnapshot.withData(ConnectionState.done, value);
    notifyListeners();
  }
}

class ScreeningScreen extends StatelessWidget {
  final String patientId;

  const ScreeningScreen({
    super.key,
    required this.patientId,
  });

  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        ChangeNotifierProvider<_Notifier>(
          create: (_) => _Notifier(patientId: patientId),
        ),
      ],
      builder: (context, _) {
        return Scaffold(
          body: Selector<_Notifier, AsyncSnapshot<_NotifierValue?>(
            selector: (context, notifier) => notifier.snapshot,
            builder: (context, snapshot, _) {
              if (snapshot.connectionState == ConnectionState.waiting) {
                return const Center(child: CircularProgressIndicator());
              }
              final _NotifierValue? value = snapshot.data;
              return Column(
                children: [
                  // Imagem A
                  if (value == null)
                    // Passo 3.1: sem arquivos locais, consultar em nuvem
                    Image.network(
                      'https://example.com/api/patients/$patientId/screening'
                    ),
                  else
                    // Passo 3.2: com arquivos locais, exibir em cache
                    FutureBuilder<Uint8List>(
                      future: value.file.readAsBytes(),
                      builder: (context, snapshot) {
                        if (snapshot.connectionState == ConnectionState.waiting) {
                          return const Center(child: CircularProgressIndicator());
                        }
                        final Uint8List bytes = snapshot.data;
                        return Image.memory(bytes);
                      }, 
                    ),
                  // Botão B
                  if (value != null)
                    // Passo 3.1
                    TextButton(
                      onPressed: () {
                        final _Notifier notifier = context.read();
                        notifier.synchronize(value);
                      },
                      child: const Text('sincronizar'),
                    ),
                ],
              );
            },
          ),
        );
      },
    );
  }
}
```

## Conclusão

O padrão de design sugerido nessa postagem é apenas uma sugestão de
arquitetura limpa e multiplataforma para gerenciar o armazenamento em nuvem
em Dart. Para uma melhor experiência do usuário, seria interessante adicionar
a este padrão uma execução em segundo plano do envio para a nuvem e de forma
automática, mas por motivos de brevidade, essa parte pode ser deixada para
uma outra postagem (ou a exercício do leitor).

Como mencionado anteriormente, utilizei este padrão em um projeto há uns meses
e funcionou consideravelmente bem, então resolvi publicar esta explicação
para auxiliar outros colegas desenvolvedores que possam se deparar com o
mesmo problema. Espero que o intuito tenha sido alcançado!
