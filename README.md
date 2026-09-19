# Introdução ao Docker

Primeira etapa do trabalho da disciplina de **Redes e Administração de Sistemas** do Curso Técnico em Informática Integrado ao Ensino Médio do IFSP, campus Campos do Jordão.

O objetivo foi instalar o Docker em uma máquina virtual com Ubuntu Server, escrever uma aplicação web em **Python** com o framework **Flask** e executá-la dentro de um **container**, acessando a página pelo navegador.

- **Ambiente:** Ubuntu Server sobre Oracle VirtualBox


---

## Sumário

1. [O que é Docker](#1-o-que-é-docker)
2. [Preparação da máquina virtual](#2-preparação-da-máquina-virtual)
3. [Instalação do Docker](#3-instalação-do-docker)
4. [Validação com hello-world](#4-validação-com-hello-world)
5. [Estrutura do projeto](#5-estrutura-do-projeto)
6. [A aplicação Flask](#6-a-aplicação-flask)
7. [O Dockerfile](#7-o-dockerfile)
8. [Construção da imagem](#8-construção-da-imagem)
9. [Execução do container](#9-execução-do-container)
10. [Testes de acesso](#10-testes-de-acesso)
11. [Evolução da aplicação](#11-evolução-da-aplicação)
12. [Comandos úteis](#12-comandos-úteis)
13. [Problemas encontrados](#13-problemas-encontrados)

---

## 1. O que é Docker

Docker é uma plataforma de código aberto que empacota uma aplicação junto de todas as suas dependências em uma unidade isolada e portável, chamada **container**.

A diferença para uma máquina virtual é o nível em que a virtualização acontece:

| Container | Máquina virtual |
|---|---|
| Compartilha o kernel do host | Tem kernel próprio |
| Leve, na casa dos MB | Pesada, na casa dos GB |
| Inicia em segundos | Inicia em minutos |
| Isolamento em nível de processo | Isolamento completo |

Três conceitos que não podem ser confundidos:

- **Imagem:** template imutável com a aplicação e suas dependências. Não executa sozinha.
- **Container:** instância em execução de uma imagem.
- **Dockerfile:** arquivo de texto com as instruções para construir uma imagem.

---

## 2. Preparação da máquina virtual

A máquina virtual roda **Ubuntu Server** no VirtualBox. Dois ajustes foram necessários:

**Rede.** Durante a instalação dos pacotes o adaptador ficou em **NAT**, que basta para a VM alcançar a internet. Depois, para acessar a aplicação pelo navegador do computador hospedeiro, o adaptador foi trocado para **Bridge**.

**Repositórios antigos.** A imagem da VM disponibilizada no laboratório já trazia arquivos de repositório desatualizados, que precisam ser removidos antes de instalar pela fonte oficial:

```bash
sudo rm /etc/apt/sources.list.d/docker.list
sudo rm /etc/apt/sources.list.d/docker.sources
sudo apt-get update
```

---

## 3. Instalação do Docker

Seguindo a [documentação oficial](https://docs.docker.com/engine/install/ubuntu/), na modalidade de instalação a partir do repositório:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io \
     docker-buildx-plugin docker-compose-plugin
```

![Instalação do Docker](imagens/instalacao-docker.png)

Foram instalados 7 pacotes, ocupando cerca de **382 MB** adicionais em disco.

> Os comandos do Docker precisam de `sudo`, ou o usuário precisa estar no grupo `docker`:
> ```bash
> sudo usermod -aG docker $USER
> ```
> É preciso sair e entrar de novo na sessão para o grupo valer.

---

## 4. Validação com hello-world

```bash
sudo docker run hello-world
```


A saída desse comando descreve exatamente o que aconteceu por trás:

1. o **cliente** contatou o **daemon**;
2. o daemon baixou a imagem `hello-world` do **Docker Hub**, porque ela não existia localmente;
3. criou um container a partir dela;
4. executou o programa e devolveu a saída para o terminal.

---

## 5. Estrutura do projeto

```
projeto-flask/
├── app/
│   ├── app.py
│   └── requirements.txt
└── Dockerfile
```



Manter o código dentro de `app/` não é só organização. Isso permite copiar **primeiro** o `requirements.txt` para a imagem e só **depois** o código, o que aproveita muito melhor o cache de camadas do Docker.

Em seguida, a imagem base foi baixada:

```bash
docker pull python:3.14-slim
```



A variante `slim` traz só o necessário para rodar Python, seguindo a boa prática de construir imagens leves.

```bash
docker image ls
```



---

## 6. A aplicação Flask

`app/app.py`, primeira versão:

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return """
    <h1>Minha primeira aplicação Flask</h1>
    <p>Aplicação executada dentro de um container Docker!</p>
    """

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Dois detalhes são decisivos para funcionar dentro de um container:

- **`host='0.0.0.0'`** faz o servidor aceitar conexões de qualquer interface de rede. Com o padrão `127.0.0.1`, a aplicação só responderia a requisições vindas de dentro do próprio container e o mapeamento de portas não teria efeito nenhum.
- **A porta 5000** precisa ser a mesma no `EXPOSE` do Dockerfile e no `-p` do `docker run`.

`app/requirements.txt`:

```
flask
```

---

## 7. O Dockerfile

```dockerfile
FROM python:3.14-slim

WORKDIR /app

COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ .

EXPOSE 5000

CMD ["python", "app.py"]
```

O que cada instrução faz:

| Instrução | Função |
|---|---|
| `FROM` | define a imagem base |
| `WORKDIR` | define o diretório de trabalho dentro da imagem |
| `COPY` | copia arquivos do host para a imagem |
| `RUN` | executa comandos durante a construção |
| `EXPOSE` | documenta a porta em que a aplicação escuta |
| `CMD` | comando executado quando o container inicia |

**Por que copiar o `requirements.txt` antes do código?** Cada instrução gera uma camada. Como as dependências mudam muito menos que o código, deixá-las em uma camada anterior faz com que alterar o `app.py` não obrigue a reinstalar o Flask a cada build. O `--no-cache-dir` do pip evita gravar arquivos temporários na camada, deixando a imagem menor.

---

## 8. Construção da imagem

```bash
docker build -t minha-flask .
```



O `-t` dá um nome à imagem e o ponto final indica o **contexto de construção**, ou seja, o diretório cujo conteúdo é enviado ao daemon.

A construção levou **9,3 segundos** e percorreu 10 etapas. A mais demorada foi a instalação do Flask pelo pip, com 7,9 segundos, que é justamente o motivo de isolá-la em uma camada própria.

---

## 9. Execução do container

```bash
docker run -d -p 5000:5000 --name meu-flask minha-flask
docker ps
```



| Opção | Função |
|---|---|
| `-d` | executa em segundo plano (detached) |
| `-p 5000:5000` | mapeia a porta do host para a porta do container |
| `--name` | dá um nome legível ao container |

O `docker ps` mostra o container com status `Up` e o mapeamento `0.0.0.0:5000->5000/tcp` ativo.

```bash
docker image ls
```



A `minha-flask` aparece com 211 MB contra 191 MB da imagem base, mas o espaço realmente ocupado a mais é de apenas **20 MB**. Todas as camadas herdadas do Python são compartilhadas fisicamente entre as duas imagens. Esse é o efeito do sistema de camadas.

---

## 10. Testes de acesso

**Dentro da VM, pelo terminal:**

```bash
curl http://localhost:5000
```


**No navegador do computador hospedeiro** (com o adaptador de rede já em modo Bridge):



---

## 11. Evolução da aplicação

Com o fluxo funcionando, a aplicação ganhou as rotas `/sobre` e `/contato`, um menu de navegação compartilhado e uma folha de estilos CSS.



Como o código está **dentro da imagem**, editar o arquivo não muda o container que já está rodando. É preciso refazer o ciclo:

```bash
docker stop meu-flask
docker rm meu-flask
docker build -t minha-flask .
docker run -d -p 5000:5000 --name meu-flask minha-flask
```

Esse segundo build foi bem mais rápido, porque só a camada de cópia do código precisou ser refeita. Todas as anteriores vieram do cache.

**Resultado:**



---

## 12. Comandos úteis

```bash
docker --version                  # versão instalada
docker ps                         # containers em execução
docker ps -a                      # todos os containers
docker image ls                   # imagens locais
docker logs meu-flask             # logs do container
docker stop meu-flask             # para o container
docker rm meu-flask               # remove o container
docker rmi minha-flask            # remove a imagem
docker exec -it meu-flask bash    # abre um shell dentro do container
```

---

## 13. Problemas encontrados

**A página não abre no navegador do hospedeiro.**
O adaptador da VM estava em NAT. Em NAT a VM alcança a internet, mas o hospedeiro não alcança a VM. A solução foi trocar para **Bridge** nas configurações de rede do VirtualBox.

**O `apt-get update` acusa conflito de repositórios.**
Sobraram arquivos antigos em `/etc/apt/sources.list.d`. Remover `docker.list` e `docker.sources` resolve.

**O container está `Up` mas nada responde na porta 5000.**
O Flask estava escutando só em `127.0.0.1`. É preciso declarar `app.run(host='0.0.0.0', port=5000)`.

**`permission denied` ao rodar o docker.**
Usar `sudo` ou adicionar o usuário ao grupo `docker` e reiniciar a sessão.

---

## Como reproduzir

```bash
git clone https://github.com/SEU-USUARIO/introducao-ao-docker.git
cd introducao-ao-docker
docker build -t minha-flask .
docker run -d -p 5000:5000 --name meu-flask minha-flask
```

Depois é só abrir `http://localhost:5000`.

---

## Referências

- [Documentação oficial do Docker](https://docs.docker.com)
- [Instalação do Docker Engine no Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Hub](https://hub.docker.com)
- [Documentação do Flask](https://flask.palletsprojects.com)
- VITALINO, J. F. N.; CASTRO, M. A. N. *Descomplicando o Docker*. Rio de Janeiro: Brasport, 2016.
