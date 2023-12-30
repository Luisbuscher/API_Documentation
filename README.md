INFORMAÇÕES IMPORTANTES:

    - O uso da API mostrado aqui é um uso simplificado para conexão rápida.
    - Portanto, algumas funções especificas podem exigir uma abordagem diferente, na qual não será encontrada aqui.
    - Importante saber o uso de socket.io para melhor entendimento.

USO DA API. Passo 1: No back-end do projeto crie um arquivo chamado ex: api.js e cole o seguinte código:

    const { json } = require("express");

    class User {

        // ATUALIZA A CARTEIRA DO USUARIO:
        async updateWalletUser(value, io, typeWallet, _idUser, _tokenUser, _tokenSystem, name_game, status, url) {
            try {
                let valueFormatedBRL = this.formatedIntValue(value); // Formata o numero para inteiro. Ex: 10.90 --> 1090.

                let arrays = {
                    userId: _idUser, // Id do usuário.
                    tokenId: _tokenUser, // Token do usuário.
                    tokenSystem: _tokenSystem, // Token do sistema.
                    typeId: 3, // Type id do serviço.
                    typeWallet: typeWallet, // Add => adicionar valores || remove => remover valores.
                    Value: valueFormatedBRL,
                    Value_type: 'balance', // Balance => saldo real || bonus => saldo do bonus.
                    Service_name: `Slot: ${name_game}`, // Nome do serviço.
                    Service_type: status, // Win => venceu || loser => perdeu || loser => stats => inicou a "partida".
                };

                await fetch(`https://${url}/rest_api_process`, {
                    method: 'POST',
                    body: JSON.stringify(arrays),
                    headers: { 'Content-Type': 'application/json' }
                })
                    .then(res =>
                        res.json()
                    ).then(json =>
                        console.log(`Aposta realizada com sucesso!`)
                    );

                this.updateWalletDisplay(io, _idUser, _tokenUser, _tokenSystem, url); // Funcao para atualizar display da carteira do usuario.
            } catch (error) {
                console.error('Erro ao atualizar carteira do usuario: ', error);
                throw error;
            }
        }

        // CONSULTA OS DADOS DO USUARIO:
        async queryData(idUser, tokenUser, _tokenSystem, url) {
            try {
                let arrays = {
                    userId: idUser,
                    tokenId: tokenUser,
                    tokenSystem: _tokenSystem,
                    typeId: 1,
                };

                return fetch(`https://${url}/rest_api_process`, {
                    method: 'POST',
                    body: JSON.stringify(arrays),
                    headers: { 'Content-Type': 'application/json' },
                })
                    .then((res) => res.json())
                    .then((json) => {
                        return json; // Retornar o valor do JSON para que possa ser usado na função chamadora.
                    });
            } catch (error) {
                console.error('Erro ao consultar dados do usuario: ', error);
                throw error;
            }
        }

        // VALIDA A APOSTA E RETIRA VALOR APOSTADO DA CARTEIRA:
        async validadeBet(value, idUser, tokenUser, tokenSystem, io, name_game, status, url) {
            try {
                value = value.toString();
                let valueFormated = value.replace('.', '');
                valueFormated = valueFormated.replace(',', '.');
                valueFormated = parseFloat(valueFormated); // Valor formatado em float EX: 509,09 => 509.09.
                let dataUser = await this.queryData(idUser, tokenUser, tokenSystem, url); // Pega os dados do usuario.
                let currentWallet = dataUser.user_pointer; // Pega o saldo atual da carteira do usuario com formato float EX: 10.82.
                currentWallet = currentWallet / 100;
                if (currentWallet >= valueFormated) {
                    // Atualiza a carteira via API do usuario e o display.
                    await this.updateWalletUser(valueFormated, io, 'remove', idUser, tokenUser, tokenSystem, name_game, status, url);
                    return valueFormated; // Retorna que a aposta foi feita com sucesso.
                } else {
                    io.emit('messageErroServer', 'Aposta invalida, por favor consulte o saldo e faça uma aposta valida!');
                    return false;
                } // Retorna que a aposta nao teve sucesso.

            } catch (error) {
                io.emit('messageErroServer', 'Ocorreu um erro ao relizar a aposta!');
                console.error('Erro ao formatar numero: ', error);
                return false;
            }
        }

        // ATUALIZA/EXIBE DISPLAY DO SALDO DO USUARIO NO FRONT-END:
        async updateWalletDisplay(io, _idUser, _tokenUser, _tokenSystem, url) {
            try {
                let dataUser = await this.queryData(_idUser, _tokenUser, _tokenSystem, url);
                let currentWallet = dataUser.user_pointer;
                currentWallet = this.formatFloatValue(currentWallet);
                io.emit('updateCurrentWallet', currentWallet); // Atualiza o display da carteira.
            } catch (error) {
                console.error('Erro ao atualizar o display da carteira do usuario: ', error);
                throw error;
            }
        }

        // CONVERTE VALOR PARA INTEIRO:
        formatedIntValue(value) {
            let valueFormatedInt = value * 100;
            return valueFormatedInt;
        }

        // CONVERTE VALOR PARA FLOAT:
        formatFloatValue(value) {
            let valueFormated = value / 100;
            return valueFormated;
        }

    }

    module.exports = User;

Passo 2: Dentro do arquivo main, importe a o arquivo criado, antes das demais coisas, ex:

    const Api = require('./script/api/api.js');
    const api = new Api();

Passo 3: Usando o socket.io, use o seguinte trecho de código padronizado para efetuar o uso da API nos jogos:

    io.on('connection', (socket) => {
        // NOME DO JOGO:
        let name_game = 'Dinossauro';

        // VARIAVEIS DOS PARAMETROS:
        let tokenRequestJogos;
        let tokenIdUser;
        let userId;
        let nativeUrl;

        // ENVIANDO PARAMETROS PARA AS DEVIDAS VARIAVEIS E ATUALIZANDO O DISPLAY DA CARTEIRA:
        socket.on('send_parameters', (parameters) => {
            const currentRoute = parameters;

            const parsedUrl = url.parse(currentRoute, true);

            const queryParameters = parsedUrl.query;
            tokenIdUser = queryParameters.tokenId_User;
            userId = queryParameters.user_id;
            nativeUrl = queryParameters.native_url;

            for (let i = 0; i < urls.length; i++) {
                if (nativeUrl == urls[i].url) {
                    tokenRequestJogos = urls[i].token;
                }
            };

            console.log(tokenIdUser + '\n' + userId + '\n' + nativeUrl);

            api.updateWalletDisplay(socket, userId, tokenIdUser, tokenRequestJogos, nativeUrl);
        });

        // INICIAR O JOGO RETIRANDO DA CARTEIRA VALOR APOSTADO:
        socket.on('play', async (valueBet) => {
            // Recebe false se ocorreu erro ao efetuar aposta ou o valor de aposta formatado (10,00) de obtiver sucesso.
            let sucessBet = await api.validadeBet(valueBet, userId, tokenIdUser, tokenRequestJogos, socket, name_game, 'stats', nativeUrl);
            if (sucessBet != false) {
                console.log(valueBet)
                // Inicia o jogo.
                socket.emit('startGame');
            } else {
                socket.emit('errorGame');
            }
        });

        // ATUALIZA A CARTEIRA COM O NOVO VALOR CASO O USUARIO TENHA VENCIDO:
        socket.on('win', async (value) => {
            api.updateWalletUser(value, socket, 'add', userId, tokenIdUser, tokenRequestJogos, name_game, 'win', nativeUrl);
        });

    });

Agora é só adaptar o front-end com o back-end usando a sua lógica para enviar os valores corretos para api.

PASSO 5: EXEMPLO DE ENVIO DO FRONT-END para o back-end. Enviando os parametros definidos no como "send_parameters":

    var url = window.location.href;
    socket.emit('send_parameters', url);
