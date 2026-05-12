# Relatório Técnico de Implementação do OAuth 2.0 no Task Manager

## Introdução

Este documento descreve a arquitetura e a implementação do protocolo OAuth 2.0 na nossa API de gestão de tarefas. O sistema foi pensado para garantir a segurança dos dados sem prejudicar a fluidez do desenvolvimento, e a sua construção assentou na integração entre o Express, o Passport e o Sequelize.

## A Escolha do OAuth 2.0 Face às Alternativas

Comparando o OAuth 2.0 com soluções como a Autenticação Básica ou os tokens JWT, as diferenças são significativas. A Autenticação Básica envia credenciais em texto, o que constitui uma falha grave de segurança. Os tokens JWT resolvem o problema do estado do servidor, mas continuam a exigir que a equipa desenvolva e mantenha sistemas de criação, encriptação e validação de passwords locais.

Ao optar pelo OAuth 2.0 com o GitHub, este problema desaparece. A gestão de credenciais é delegada a um provedor externo de confiança, pelo que o nosso sistema deixa de guardar palavras-passe, eliminando à partida o risco de fugas de dados sensíveis.

## Configuração e Estratégia do Passport

A implementação começou pela configuração do Passport.js no ficheiro `config/auth.js`. Foi instanciada a `GitHubStrategy` com as chaves de acesso geradas no portal de developers do GitHub.

**Exemplo de implementação da estratégia:**

```javascript
const passport = require("passport");
const GitHubStrategy = require("passport-github2").Strategy;

const passportOptions = {
    clientID: GITHUB_CLIENT_ID,
    clientSecret: GITHUB_CLIENT_SECRET,
    callbackURL: "http://localhost:3000/auth/github/callback"
};

passport.use(new GitHubStrategy(passportOptions,
    function (accessToken, refreshToken, profile, done) {
        profile.token = accessToken;
        return done(null, profile);
    }
));
```

Quando o GitHub devolve o perfil do utilizador, extraímos o token e passamos o objeto para a sessão do Express através dos métodos `serializeUser` e `deserializeUser`, ficando o utilizador acessível em toda a API.

## Implementação do Fluxo de Rotas

Com o Passport configurado, integramos o fluxo de redirecionamento no ficheiro principal `index.js`, através de duas rotas que controlam a entrada dos utilizadores.

**Exemplo de implementação das rotas de autenticação:**

```javascript
app.get("/auth/github", passport.authenticate("github", { scope: ["user:email"] }));

app.get("/auth/github/callback",
    passport.authenticate("github", { failureRedirect: "/login" }),
    function (req, res) {
        console.log("Utilizador autenticado com sucesso:", req.user.username);
        res.redirect("/");
    }
);
```

A primeira rota encaminha o utilizador para o ecrã de autorização do GitHub, pedindo acesso ao email. A segunda rota recebe o redirecionamento de volta, processa o código de troca e regista o utilizador, colocando os seus dados no objeto `req.user`.

## Defesa de Rotas e Middleware

Autenticar utilizadores não basta se as rotas da API ficarem desprotegidas. Para isso foi criado um middleware no ficheiro `authMiddleware.js`, que interceta os pedidos antes de chegarem aos controladores.

**Exemplo de implementação do bloqueio de rotas:**

```javascript
const auth = function (req, res, next) {
    if (req.isAuthenticated()) {
        console.log("Pedido intercetado e validado do utilizador ID:", req.user.id);
        return next();
    }
    res.status(401).json({ erro: "Autenticacao necessaria. Acesso negado." });
};

module.exports = auth;
```

Em cada pedido às rotas de projetos ou tarefas, a função `isAuthenticated` verifica a sessão. Se o utilizador não tiver uma sessão válida, o pedido é bloqueado com um erro 401, evitando que processamento não autorizado consuma recursos da base de dados.

## Isolamento de Dados com Sequelize

O último requisito foi garantir a separação dos recursos: cada utilizador só pode aceder e modificar os seus próprios projetos. Esta regra é aplicada injetando o ID do GitHub nas cláusulas de pesquisa e modificação do Sequelize.

**Exemplo de implementação no controlador de projetos:**

```javascript
exports.getAllProjects = async (req, res) => {
    try {
        const projects = await Project.findAll({
            where: { UserId: req.user.id },
            include: [{ model: Task }]
        });
        res.status(200).json(projects);
    } catch (error) {
        res.status(500).json({ erro: "Falha ao listar os projetos" });
    }
};
```

O ponto-chave está na cláusula `where: { UserId: req.user.id }`. Como o `req.user.id` vem diretamente da sessão gerida pelo Passport, qualquer tentativa de manipulação por parte do cliente no corpo do pedido é ineficaz. O Sequelize aplica este filtro na base de dados, devolvendo e alterando apenas os registos que pertencem ao utilizador autenticado.

## Conclusão

A implementação resultou numa API segura e funcional, que combina o Express, o Passport e o Sequelize. O fluxo OAuth 2.0 simplificou a gestão de credenciais e permitiu que o esforço de desenvolvimento se concentrasse no isolamento dos dados entre utilizadores. O sistema final é rápido, fiável e devidamente protegido.
