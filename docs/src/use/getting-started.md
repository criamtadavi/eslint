const { Client, GatewayIntentBits, SlashCommandBuilder, REST, Routes, ChannelType, ButtonStyle, ActionRowBuilder, ButtonBuilder, EmbedBuilder } = require('discord.js');
const config = require('./config.json'); // Carrega o arquivo config.json

// Substitua as variáveis pelos valores do arquivo config.json
const { TOKEN, GUILD_ID, CLIENT_ID, TICKET_CATEGORY_ID, STAFF_ROLE_ID } = config;

const client = new Client({ intents: [GatewayIntentBits.Guilds] });

// Registro do comando /ticket
const commands = [
    new SlashCommandBuilder()
        .setName('ticket')
        .setDescription('Abre o painel do sistema de tickets'),
].map(command => command.toJSON());

const rest = new REST({ version: '10' }).setToken(TOKEN);

(async () => {
    try {
        console.log('Iniciando o registro de comandos...');
        await rest.put(
            Routes.applicationGuildCommands(CLIENT_ID, GUILD_ID),
            { body: commands }
        );
        console.log('Comandos registrados com sucesso!');
    } catch (error) {
        console.error('Erro ao registrar comandos:', error);
    }
})();

client.once('ready', () => {
    console.log(`Bot está online como ${client.user.tag}`);
});

let ticketCounter = 1; // Contador de tickets

client.on('interactionCreate', async (interaction) => {
    if (interaction.isCommand() && interaction.commandName === 'ticket') {
        try {
            await interaction.reply({
                embeds: [
                    {
                        color: 0x00ff00,
                        title: '🎫 Bem-vindo(a) ao sistema de tickets!',
                        description: `
**Informe o que você precisa.** Tickets que não forem diretos ao ponto serão **fechados**.

**Instruções:**
- Para denúncias, é necessário enviar um vídeo (pode ser enviado no YouTube).
- Para prints, use Imgur ou outra plataforma similar.

Escolha uma categoria abaixo para atendimento:
            `,
                        image: {
                            url: 'https://media.discordapp.net/attachments/897447155743223819/1097129475893887036/wallpaper.png',
                        },
                        fields: [
                            {
                                name: '📋 Atendimento',
                                value: 'Precisa de ajuda? Selecione uma das opções abaixo.',
                            },
                            {
                                name: '⏰ Horário de Atendimento',
                                value: 'Atendimento de segunda a sexta. Fora do horário, não garantimos respostas imediatas.',
                            },
                            {
                                name: '🌟 Deseja ser VIP?',
                                value: 'Confira nossa loja usando o comando `/loja`. Entrega automática e descontos via PIX!',
                            },
                        ],
                        footer: {
                            text: 'Obrigado por usar nosso sistema de tickets!',
                        },
                    },
                ],
                components: [
                    {
                        type: 1,
                        components: [
                            {
                                type: 3,
                                custom_id: 'ticket_select',
                                placeholder: 'Escolha a categoria do seu atendimento',
                                options: [
                                    {
                                        label: 'Denúncias',
                                        description: 'Reportar jogadores ou infrações',
                                        value: 'denuncias',
                                    },
                                    {
                                        label: 'Ajuda com Compras',
                                        description: 'Problemas ou dúvidas sobre compras',
                                        value: 'compras',
                                    },
                                    {
                                        label: 'Dúvidas sobre Corpo ou Facções',
                                        description: 'Suporte relacionado a corporações e facções',
                                        value: 'corpo_fac',
                                    },
                                    {
                                        label: 'Outros',
                                        description: 'Suporte para outros assuntos gerais',
                                        value: 'outros',
                                    },
                                ],
                            },
                        ],
                    },
                ],
            });
        } catch (error) {
            console.error('Erro ao responder ao comando:', error);
            if (!interaction.replied) {
                await interaction.reply({ content: 'Ocorreu um erro ao processar seu pedido.', ephemeral: true });
            }
        }
    }

    if (interaction.isSelectMenu() && interaction.customId === 'ticket_select') {
        const category = interaction.values[0];
        const channelName = `ticket-${ticketCounter}`;

        try {
            const guild = interaction.guild;
            const ticketChannel = await guild.channels.create({
                name: channelName,
                type: ChannelType.GuildText,
                parent: TICKET_CATEGORY_ID,
                topic: `Ticket criado por ${interaction.user.tag} para ${category}`,
                permissionOverwrites: [
                    {
                        id: guild.roles.everyone.id,
                        deny: ['ViewChannel'],
                    },
                    {
                        id: interaction.user.id,
                        allow: ['ViewChannel', 'SendMessages', 'ReadMessageHistory'],
                    },
                ],
            });

            await interaction.reply({
                content: `Seu ticket foi criado com sucesso: ${ticketChannel}`,
                ephemeral: true,
            });

            // Incrementar o contador de tickets
            ticketCounter++;

            // Enviar mensagem no canal do ticket
            const embed = new EmbedBuilder()
                .setColor(0x00ff00)
                .setTitle(`🎫 Categoria: ${category}`)
                .setDescription(
                    'Informe sua dúvida e/ou preencha sua denúncia se for o caso. Tickets que não forem diretos ao ponto serão fechados sem aviso prévio!'
                );

            const closeButton = new ButtonBuilder()
                .setCustomId('close_ticket')
                .setLabel('Fechar Ticket')
                .setStyle(ButtonStyle.Danger);

            const fillReportButton = new ButtonBuilder()
                .setCustomId('fill_report')
                .setLabel('Preencher Denúncia')
                .setStyle(ButtonStyle.Success);

            const assistButton = new ButtonBuilder()
                .setCustomId('assist_ticket')
                .setLabel('Atender')
                .setStyle(ButtonStyle.Primary);

            const row = new ActionRowBuilder().addComponents(closeButton, fillReportButton, assistButton);

            await ticketChannel.send({
                embeds: [embed],
                components: [row],
            });
        } catch (error) {
            console.error('Erro ao criar o canal do ticket:', error);
            await interaction.reply({
                content: 'Ocorreu um erro ao criar o seu ticket. Por favor, tente novamente mais tarde.',
                ephemeral: true,
            });
        }
    }

    if (interaction.isButton()) {
        switch (interaction.customId) {
            case 'close_ticket':
                await interaction.reply('O ticket será fechado em breve.');
                await interaction.channel.delete();
                break;
            case 'fill_report':
                await interaction.reply({
                    content: 'Preencha sua denúncia utilizando o formulário abaixo:\n\n' +
                             '**ID do denunciante**: Seu ID no canto inferior do Discord.\n' +
                             '**ID do acusado**: ID da pessoa que você está denunciando.\n' +
                             '**Provas do acontecimento**: Link para um vídeo (YouTube) ou imagem (Imgur).\n' +
                             '**Descrição**: Explique o que aconteceu.',
                    ephemeral: true
                });
                break;
            case 'assist_ticket':
                // Verifica se o usuário tem o cargo de Staff
                const member = interaction.guild.members.cache.get(interaction.user.id);
                if (member.roles.cache.has(STAFF_ROLE_ID)) {
                    try {
                        const ticketCreatorMatch = interaction.channel.topic?.match(/Ticket criado por (.+?) para/);
                        if (ticketCreatorMatch) {
                            const ticketCreatorTag = ticketCreatorMatch[1];
                            const user = interaction.guild.members.cache.find(m => m.user.tag === ticketCreatorTag)?.user;

                            if (user) {
                                await user.send(`Seu ticket está sendo atendido por ${interaction.user.tag}.`);
                            }
                        }

                        // Envia mensagem no canal do ticket
                        await interaction.channel.send(`Este ticket está sendo atendido por ${interaction.user.tag}.`);
                        await interaction.reply({ content: 'Você iniciou o atendimento a este ticket.', ephemeral: true });
                    } catch (error) {
                        console.error('Erro ao processar atendimento:', error);
                        await interaction.reply({ content: 'Erro ao iniciar
