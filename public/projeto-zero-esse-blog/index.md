---
title: "Projeto zero esse blog"
date: '2024-06-15'
spoiler: "terraform, cloudflare e next.js"
---
## Apresentação

```js
<p className="text-2xl font-sans text-purple-400 dark:text-purple-500">
  <i>Fala rapeize!!!</i>!
</p>
```

Neste tutorial, vamos configurar e realizar o nosso primeiro deploy de um projeto no Cloudflare Pages usando Terraform e GitHub. Vamos abordar os passos necessários para criar um projeto Pages, configurá-lo com uma fonte GitHub e realizar o primeiro commit para iniciar o build e deployment.

## Pré-requisitos
- Conta no Cloudflare
- Conta no GitHub 
- [Terraform instalado](https://developer.hashicorp.com/terraform/install) em sua máquina 

---
## Repositório de código

Procurando por personal blog no GitHub, me deparei com: [overreacted.io](https://github.com/gaearon/overreacted.io), que me encantou de primeira pelo design e estilo simples. Forkei ele e dei o nome do meu site `devopslaboratory-com`.

## Tokens

Por conveniência, uso a minha [Global API Key](https://dash.cloudflare.com/profile/api-tokens). Guarde o valor, pois vamos usar para o `api_key`.

É preciso ter um website cadastrado, porque o `account_id` é um recurso necessário para o deploy. Após o seu website estar configurado seguindo os passos da Cloudflare, em Overview > Account ID, guarde o valor.

## GitHub e Cloudflare

A forma mais simples que encontrei de fazer a associação com o GitHub é seguindo a documentação. Retirei um trecho:

1. Faça login no [Cloudflare dashboard](https://dash.cloudflare.com/) e selecione sua conta.
2. Na Home da conta, selecione Workers & Pages.
3. Selecione Create application > Pages > Connect to Git.

## main.tf 

Crie um arquivo `main.tf` e adicione a configuração do provedor do Cloudflare:

```hcl
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "cloudflare" {
  api_token = "seu-api-token"
}
```

Substitua pelo `api_token` que estava guardado.

Inicie o provider com o comando `terraform init`:

```shell
terraform init   

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of cloudflare/cloudflare from the dependency lock file
- Using previously-installed cloudflare/cloudflare v4.37.0

Terraform has been successfully initialized!

(...)
```

## Configurar o Projeto Pages com Build

No mesmo arquivo `main.tf`, adicione a configuração para criar o projeto Pages com a configuração de build:

```hcl
resource "cloudflare_pages_project" "basic_project" {
  account_id        = "account ID" # 1.
  name              = "guelo"      # 2. 
  production_branch = "main"

  source {
    type = "github"
    config {
      owner             = "fbrunoviana"          # 3.
      repo_name         = "devopslaboratory-com" # 4.
      production_branch = "main"
    }
  }

  build_config {
    build_command   = "npm run build" # 5.
    destination_dir = "out"
  }
}
```

Explicação:

- Seu `account_id` guardado nos passos anteriores.
- Nome do seu projeto, para testes uso "guelo".
- Seu usuário do GitHub.
- O nome do seu repositório.
- Comando para buildar o código.

### Correr pro abraço

Vamos planejar o nosso deploy:

```shell
terraform plan -out=plan

(...)
Plan: 1 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Saved the plan to: plan

To perform exactly these actions, run the following command to apply:
    terraform apply "plan"
```

Então execute:

```shell
terraform apply "plan"
```

### Verificar o Deployment no Cloudflare

Após aplicar a configuração, acesse o painel do Cloudflare.

Navegue até a seção "Pages" e verifique se o projeto foi criado corretamente.

Verifique se o primeiro build/deployment foi disparado.

### Commit 

É necessário fazer alguma mudança, aproveite esse commit para mudar coisas como nome, GitHub, fotos.

Realize um commit e faça o push para o repositório especificado:

```bash
git add .
git commit -m "Primeiro commit para deployment no Cloudflare Pages"
git push origin main
```

## Conclusão

Assim, o seu código foi deployado com sucesso no Cloudflare. O codigo completo fica em: `https://github.com/fbrunoviana/cloudflare-page.git`