# Integração Jerakia e Vault

- Neste procedimento foi utilizado uma VM com o Vault e o Jerakia já instalados e configurados, conforme documentações anteriores.

  - CentOS 7 (Jerakia 2.0 e Vault 0.7.3). secret.example.local;
  - Caso o Firewall esteja habilitado não esquecer de liberar as portas do Vault (8200/tcp);

- Primeiro precisamos ajustar o vault para trabalhar com modelo de autenticação AppRole, habilitar o backend transport que é usado pelo Jerakia para descriptografar e criptografar por demanda.

#### 1. Precisamos realizar o login no vault para poder realizar as configurações, para isso seguimos os comandos abaixo.

   - O token foi gerado anteriormente na instalação do Vault.
   - A variável do vault é o nome do servidor em que o Vault está instalado.

```shell
export VAULT_ADDR=http://secret.example.local:8200

vault auth
Token (will be hidden): 

Successfully authenticated! You are now logged in.
token: d0151df4-8a35-583f-7f74-2d4ec08485c5
token_duration: 0
token_policies: [root]
```

#### 2. Vamos habilitar os backends necessários para trabalhar com o Jerakia conforme abaixo. Será habilitado o backend de segredo transit.

```shell
vault mount transit
```

#### 3. Após montar o transit, precisamos criar o caminho com a chave que será utilizada para criptografar e decriptografar. Neste exemplo iremos criar a key Jerakia.

```shell
vault write -f transit/keys/jerakia
```

#### 4. Por questões de segurança iremos criar uma política de permissionamento para que os tokens criados pelo AppRole tenham permissão somente para criptografar e descriptografar somente no path/key jerakia, vamos criar um arquivo temporário que será utilizado para criar a permissão. Arquivo criado /tmp/jerakia_policy.hcl com o conteúdo abaixo.

```shell
path "transit/decrypt/jerakia" {
  policy = "write"
}
 
path "transit/encrypt/jerakia" {
  policy = "write"
}
```

#### 5. Após a criar o arquivo da política vamos criar a permissão no Vault.

```shell
vault policy-write jerakia /tmp/jerakia_policy.hcl
```

#### 6. Após criar a política vamos habilitar o backend de AppRole e configurar as permissões para geração dos role-id’s.

```shell
vault auth-enable approle
```

  - Após a habilitação vamos configurar as policies do AppRole para geração dos tokens dinâmicos, é importante lembrar que a os tokens gerados pelo jerakia tem que ter acesso ao backend transit configurando anteriormente com a key jerakia.

#### 7. Abaixo iremos criar uma regra com o nome de jerakia, fazendo com que os token gerados tenham tempo de vida de 10 min e tendo a permissão para acessar o caminho de criptografia e descriptografia criados anteriormente.

```shell
vault write auth/approle/role/jerakia token_ttl=10m policies=jerakia
```

#### 8. Agora iremos criar um role-id para o jerakia. É muito importante armazenar o role id e a senha, caso seja perdido, não é possível reler a senha criada.

```shell
vault read auth/approle/role/jerakia/role-id
```

  - O retorno esperado do comando acima é similar o abaixo.
  
```shell
Key 	Value
--- 	-----
role_id 4d556187-9c7b-aa41-9071-6375e8264a71
```

#### 9. Após criar o role-id precisamos criar as senhas com o comando abaixo.

  - O retorno esperado do comando é similar ao abaixo.

```shell
Key                 Value
---                 -----
secret_id           02c32b87-2a70-22b6-b07d-a36e5f4d0594
secret_id_accessor  cba4a390-b5c9-9734-b191-4c1f590b17c7
```

#### 10. Após criar as roles e os secrets, vamos configurar o jerakia para utilizar o Vault. Para realizar esse procedimentos devemos configurar no jerakia.yaml o campo encryption conforme abaixo. /etc/jerakia/jerakia.yaml.

  - OBS: Caso esteja usando ssl, ajustar conforme necessário.
  
```shell
encryption:
  provider: vault
  vault_addr: http://secret.example.local:8200
  vault_use_ssl: false
  vault_role_id: f43b0557-d485-5223-f2b2-75135391cfe5
  vault_secret_id: 8a2fa99c-7811-5e65-a74a-8ab2ba9b6389
  vault_keyname: jerakia
```

#### 11. Após configurar o Jerakia para utilizar o Vault como Encryption as a Service. Podemos rodar os comandos abaixo, para criptografar e descriptografar.

```shell
jerakia secret encrypt MySup3rs3cret
```

#### 12. Caso a criptografia do segredo tenha sido executada com sucesso, teremos um retorno similar ao que temos abaixo.

```shell
vault:v1:whpW4FtG9ZdRCZIAMAFnLO432Edkg0eiW+vHsT+eY88kbnsXNr57UiU=
```

#### 13. Para descriptografar pegamos o resultado do comando da criptografia e usamos o comando abaixo para descriptografar.

```shell
jerakia secret decrypt vault:v1:whpW4FtG9ZdRCZIAMAFnLO432Edkg0eiW+vHsT+eY88kbnsXNr57UiU=
```

  - E teremos o retorno do que criptografamos anteriormente.

```shell
MySup3rs3cret
```

- OBS: A cifra gerada com o comando da criptografia só pode ser descriptografado pela mesma key que realizou a criptografia.

- Documentações Utilizadas

  - https://www.vaultproject.io/docs/auth/approle.html
  - http://www.craigdunn.org/2017/04/managing-puppet-secrets-with-jerakia-and-vault/
