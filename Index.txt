const { Client, GatewayIntentBits, SlashCommandBuilder, REST, Routes, EmbedBuilder } = require('discord.js');
require('dotenv').config();

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
  ]
});

// ==================== HELPER ====================
function generateKey(length = 16) {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let key = '';
  for (let i = 0; i < length; i++) {
    key += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  // Format dengan dash setiap 4 karakter
  return key.match(/.{1,4}/g).join('-');
}

function getRandomDuration() {
  // Acak antara 24 - 48 jam (1-2 hari)
  return Math.floor(Math.random() * (48 - 24 + 1)) + 24;
}

function formatExpiry(date) {
  return date.toLocaleDateString('id-ID', {
    day: 'numeric',
    month: 'short',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  });
}

// ==================== REGISTER SLASH COMMAND ====================
const commands = [
  new SlashCommandBuilder()
    .setName('generatekey')
    .setDescription('Buat key acak untuk akses sistem')
    .addStringOption(option =>
      option.setName('durasi')
        .setDescription('Durasi key dalam jam, atau ketik "random" untuk 1-2 hari acak')
        .setRequired(false)
    )
    .addStringOption(option =>
      option.setName('label')
        .setDescription('Label untuk key (opsional)')
        .setRequired(false)
    )
].map(cmd => cmd.toJSON());

const rest = new REST({ version: '10' }).setToken(process.env.DISCORD_TOKEN);

(async () => {
  try {
    console.log('🔄 Mendaftarkan slash command...');
    await rest.put(
      Routes.applicationCommands(process.env.CLIENT_ID),
      { body: commands }
    );
    console.log('✅ Slash command terdaftar.');
  } catch (error) {
    console.error('❌ Gagal mendaftarkan command:', error);
  }
})();

// ==================== EVENT READY ====================
client.once('ready', () => {
  console.log(`🤖 Bot ${client.user.tag} siap!`);
});

// ==================== HANDLE SLASH COMMAND ====================
client.on('interactionCreate', async interaction => {
  if (!interaction.isChatInputCommand()) return;

  if (interaction.commandName === 'generatekey') {
    const durasiInput = interaction.options.getString('durasi') || 'random';
    const label = interaction.options.getString('label') || 'Key Generated';
    
    let durationHours;
    if (durasiInput.toLowerCase() === 'random') {
      durationHours = getRandomDuration();
    } else {
      const parsed = parseInt(durasiInput);
      if (isNaN(parsed) || parsed <= 0) {
        return interaction.reply({ content: '❌ Durasi harus berupa angka positif (dalam jam) atau "random".', ephemeral: true });
      }
      durationHours = parsed;
    }

    const key = generateKey();
    const expiryDate = new Date(Date.now() + durationHours * 60 * 60 * 1000);
    const formattedExpiry = formatExpiry(expiryDate);

    // Embed rapi
    const embed = new EmbedBuilder()
      .setColor(0x6c5ce7)
      .setTitle('🔑 Key Berhasil Dibuat')
      .addFields(
        { name: '📋 Key', value: `\`${key}\``, inline: false },
        { name: '⏳ Berlaku', value: `${durationHours} jam`, inline: true },
        { name: '📅 Kadaluarsa', value: formattedExpiry, inline: true },
        { name: '🏷️ Label', value: label, inline: false }
      )
      .setFooter({ text: 'Gunakan key ini di halaman login untuk mengakses sistem.' })
      .setTimestamp();

    await interaction.reply({ embeds: [embed], ephemeral: false });
  }
});

// ==================== LOGIN BOT ====================
client.login(process.env.DISCORD_TOKEN);
