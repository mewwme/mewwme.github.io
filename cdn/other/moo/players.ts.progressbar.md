import { KazagumoPlayer, KazagumoTrack } from "better-kazagumo";
import { Manager } from "../../manager.js";
import {
  AttachmentBuilder,
  ButtonInteraction,
  ComponentType,
  SnowflakeGenerateOptions,
  TextChannel,
  User,
} from "discord.js";
import { EmbedBuilder } from "discord.js";
import formatduration from "../../structures/FormatDuration.js";
import { QueueDuration } from "../../structures/QueueDuration.js";
import { musicCard } from "musicard";
import {
  playerRowOne,
  playerRowOneEdited,
  playerRowTwo,
} from "../../functions/playerControlButton.js";
import { replyInteraction } from "../../functions/replyInteraction.js";
import { KazagumoLoop } from "../../types/Lavalink.js";

export default async (
  client: Manager,
  player: KazagumoPlayer,
  track: KazagumoTrack // Explicitly specifying the type
) => {
  if (!client.is_db_connected)
    return client.logger.warn(
      "The database is not yet connected so this event will temporarily not execute. Please try again later!"
    );

  const guild = client.guilds.cache.get(player.guildId);
  client.logger.info(`Player Started in @ ${guild!.name} / ${player.guildId}`);

  let Control = await client.db.get(`control.guild_${player.guildId}`);
  if (!Control) {
    await client.db.set(`control.guild_${player.guildId}`, "enable");
    Control = client.db.get(`control.guild_${player.guildId}`);
  }

  if (!player) return;

  /////////// Update Music Setup ///////////

  await client.UpdateQueueMsg(player);

  /////////// Update Music Setup ///////////

  const channel = client.channels.cache.get(player.textId) as TextChannel;
  if (!channel) return;

  let data = await client.db.get(`setup.guild_${channel.guild.id}`);
  if (data && player.textId === data.channel.id) return;

  let guildModel = await client.db.get(`language.guild_${channel.guild.id}`);
  if (!guildModel) {
    guildModel = await client.db.set(
      `language.guild_${channel.guild.id}`,
      "en"
    );
  }

  const language = guildModel;

  const song = player.queue.current;
  const position = player.shoukaku.position;

  const TotalDuration = QueueDuration(player);

  if (client.websocket && client.config.features.WEB_SERVER.websocket.enable) {
    let webqueue = [];

    player.queue.forEach((track) => {
      webqueue.push({
        title: track.title,
        uri: track.uri,
        length: track.length,
        thumbnail: track.thumbnail,
        author: track.author,
        requester: track.requester, // Just case can push
      });
    });

    webqueue.unshift({
      title: song!.title,
      uri: song!.uri,
      length: song!.length,
      thumbnail: song!.thumbnail,
      author: song!.author,
      requester: song!.requester,
    });

    if (client.websocket && client.config.features.WEB_SERVER.websocket.enable)
      client.websocket.send(
        JSON.stringify({
          op: "player_start",
          guild: player.guildId,
          current: {
            title: song!.title,
            uri: song!.uri,
            length: song!.length,
            thumbnail: song!.thumbnail,
            author: song!.author,
            requester: song!.requester,
          },
        })
      );

    if (!client.sent_queue.get(player.guildId)) {
      client.websocket.send(
        JSON.stringify({
          op: "player_queue",
          guild: player.guildId,
          queue: webqueue || [],
        })
      );
      client.sent_queue.set(player.guildId, true);
    }
  }

  if (Control === "disable") return;

  const card = new musicCard()
    .setName(String(song?.title))
    .setAuthor(String(song?.author))
    .setColor("auto")
    .setTheme("classic")
    .setBrightness(50)
    .setProgress(0)
    .setThumbnail(
      track.thumbnail
        ? track.thumbnail
        : `https://img.youtube.com/vi/${track.identifier}/hqdefault.jpg`
    )
    .setRequester((song?.requester as User).username);

  const cardBuffer = await card.build();

  const attachment = new AttachmentBuilder(cardBuffer, {
    name: "moo.png",
  });

  const embeded = new EmbedBuilder()
    //    .setAuthor({
    //      name: client.i18n.get(language, "player", "track_title"),
    //      iconURL: client.i18n.get(language, "player", "track_icon"),
    //    })
    //    .setDescription(
    //      `**[${track.title}](${track.uri})**\`${formatduration(song!.length)}\`•${
    //        song!.requester
    //      }`
    //    )
    .setColor(client.color)
    .setImage("attachment://moo.png");
  //    .setFooter({
  //      text: `${client.i18n.get(language, "player", "queue_title")} ${
  //        player.queue.length + 1
  //      }`,
  //    });

  const playing_channel = client.channels.cache.get(
    player.textId
  ) as TextChannel;

  const nplaying = await playing_channel.send({
    embeds: client.config.bot.SAFE_PLAYER_MODE ? [embeded] : [],
    components: [playerRowOne, playerRowTwo],
    files: client.config.bot.SAFE_PLAYER_MODE ? [attachment] : [],
  });

  client.nplaying_msg.set(player.guildId, nplaying.id);

  const collector = nplaying.createMessageComponentCollector({
    componentType: ComponentType.Button,
    filter: (message) => {
      if (
        message.guild!.members.me!.voice.channel &&
        message.guild!.members.me!.voice.channelId ===
          message.member!.voice.channelId
      )
        return true;
      else {
        message.reply({
          content: `${client.i18n.get(language, "player", "join_voice")}`,
          ephemeral: true,
        });
        return false;
      }
    },
    time: song!.length,
  });

  let progressInterval: NodeJS.Timeout | null = null;
  let currentProgress = 0;
  let isPaused = false;
  let lastProgressUpdateTime: number | null = null;
  let progressUpdateInterval = 10 * 1000;

  async function updateProgress(
    card: musicCard,
    song: any,
    player: KazagumoPlayer,
    playing_channel: any
  ) {
    const songDuration = song.length;
    const totalUpdates = Math.ceil(songDuration / progressUpdateInterval);

    if (!player.paused) {
      if (!progressInterval && !isPaused) {
        progressInterval = setInterval(async () => {
          if (currentProgress < totalUpdates) {
            const progressPercentage = (currentProgress / totalUpdates) * 100;
            card.setProgress(progressPercentage);
            const updatedCardBuffer = await card.build();

            const updatedAttachment = new AttachmentBuilder(updatedCardBuffer, {
              name: "mewwme.png",
            });

            embeded.setImage("attachment://mewwme.png");

            const nplaying = await playing_channel.messages.fetch(
              client.nplaying_msg.get(player.guildId)
            );

            try {
              await nplaying.edit({
                embeds: [embeded],
                components: [playerRowOne, playerRowTwo],
                files: [updatedAttachment],
              });
            } catch (error) {
              console.error(
                "error updating progress because the player has been ended",
              );
            }
            currentProgress++;
          } else {
            if (progressInterval !== null) {
              clearInterval(progressInterval);
              progressInterval = null;
            }
          }
        }, progressUpdateInterval);
      } else if (lastProgressUpdateTime !== null) {
        const timeElapsedSinceUpdate = Date.now() - lastProgressUpdateTime;
        const updatesSinceLastUpdate = Math.floor(
          timeElapsedSinceUpdate / progressUpdateInterval
        );
        currentProgress += updatesSinceLastUpdate;
        lastProgressUpdateTime = Date.now();
      }
    } else {
      // Ketika lagu dijeda, berhenti memperbarui progress
      if (progressInterval !== null) {
        clearInterval(progressInterval);
        progressInterval = null;
      }
      progressInterval = null;
      lastProgressUpdateTime = null; // Hapus waktu terakhir saat update
    }
  }

  updateProgress(card, song, player, playing_channel);

  collector.on(
    "end",
    async (collected: ButtonInteraction[], reason: string) => {
      if (reason === "time") {
        if (progressInterval !== null) {
          clearInterval(progressInterval);
          progressInterval = null;
        }
      } else {
        // Cek jika koleksi memiliki tombol "stop" atau "skip"
        const stopButton = collected.find(
          (button) => button.customId === "stop"
        );
        const skipButton = collected.find(
          (button) => button.customId === "skip"
        );

        if (stopButton) {
          // Hentikan interval progress saat tombol "stop" diklik
          if (progressInterval !== null) {
            clearInterval(progressInterval);
            progressInterval = null;
          }
        } else if (skipButton) {
          // Hentikan interval progress saat tombol "skip" diklik
          if (progressInterval !== null) {
            clearInterval(progressInterval);
            progressInterval = null;
          }
        } else {
          // Jika musik telah berakhir tetapi tidak karena tombol "stop" atau "skip", hapus embed
          try {
            await nplaying.delete();
          } catch (error) {
            console.error("error deleting embed because not found/already deleted");
          }
        }
      }
    }
  );

  collector.on("collect", async (message: ButtonInteraction) => {
    const id = message.customId;
    if (id === "pause") {
      if (!player) {
        collector.stop();
      }
      player.pause(!player.paused);
      const uni = player.paused
        ? `${client.i18n.get(language, "player", "switch_pause")}`
        : `${client.i18n.get(language, "player", "switch_resume")}`;

      isPaused = player.paused;

      if (player.paused) {
        if (progressInterval !== null) {
          clearInterval(progressInterval);
          progressInterval = null;
        }
        // Clear progress update and keep track of the paused time
        lastProgressUpdateTime = null;

        nplaying.edit({
          embeds: client.config.bot.SAFE_PLAYER_MODE ? [embeded] : [],
          components: [playerRowOneEdited, playerRowTwo],
        });
      } else {
        if (progressInterval !== null) {
          clearInterval(progressInterval);
          progressInterval = null;
        }
        // Restart progress update from the paused time
        lastProgressUpdateTime =
          Date.now() - currentProgress * progressUpdateInterval;
        updateProgress(card, song, player, playing_channel);

        nplaying.edit({
          embeds: client.config.bot.SAFE_PLAYER_MODE ? [embeded] : [],
          components: [playerRowOne, playerRowTwo],
        });
      }

      if (
        client.websocket &&
        client.config.features.WEB_SERVER.websocket.enable
      )
        client.websocket.send(
          JSON.stringify({
            op: player.paused ? 3 : 4,
            guild: player.guildId,
          })
        );

      await replyInteraction(
        client,
        message,
        `${client.i18n.get(language, "player", "pause_msg", {
          pause: uni,
        })}`
      );
    } else if (id === "skip") {
      if (!player) {
        collector.stop();
      }
      player.skip();
      // Hentikan interval progress jika sedang berjalan saat melompati lagu
      if (progressInterval !== null) {
        clearInterval(progressInterval);
        progressInterval = null;
      }

      if (
        client.websocket &&
        client.config.features.WEB_SERVER.websocket.enable
      )
        client.websocket.send(
          JSON.stringify({
            op: "skip_track",
            guild: player.guildId,
          })
        );

      await replyInteraction(
        client,
        message,
        `${client.i18n.get(language, "player", "skip_msg")}`
      );
    } else if (id === "stop") {
      if (!player) {
        collector.stop();
      }
      // Hentikan interval progress saat tombol "stop" diklik
      if (progressInterval !== null) {
        clearInterval(progressInterval);
        progressInterval = null;
      }

      if (
        client.websocket &&
        client.config.features.WEB_SERVER.websocket.enable
      )
        client.websocket.send(
          JSON.stringify({
            op: "player_destroy",
            guild: player.guildId,
          })
        );

      player.destroy();

      await replyInteraction(
        client,
        message,
        `${client.i18n.get(language, "player", "stop_msg")}`
      );
    } else if (id === "shuffle") {
      if (!player) {
        collector.stop();
      }
      player.queue.shuffle();

      await replyInteraction(
        client,
        message,
        `${client.i18n.get(language, "player", "shuffle_msg")}`
      );
    } else if (id === "loop") {
      if (!player) {
        collector.stop();
      }
      const loop_mode = {
        none: "none",
        track: "track",
        queue: "queue",
      };
      // Hentikan interval progress saat tombol "stop" diklik
      if (progressInterval !== null) {
        clearInterval(progressInterval);
        progressInterval = null;
      }

      if (player.loop === "queue") {
        player.setLoop(KazagumoLoop.none);

        return await replyInteraction(
          client,
          message,
          `${client.i18n.get(language, "music", "unloopall")}`
        );
      } else if (player.loop === "none") {
        player.setLoop(KazagumoLoop.queue);
        return await replyInteraction(
          client,
          message,
          `${client.i18n.get(language, "music", "loopall")}`
        );
      }
    } else if (id === "volup") {
      if (!player) {
        collector.stop();
      }

      const reply_msg = `${client.i18n.get(language, "player", "volup_msg", {
        volume: `${player.volume * 100 + 10}`,
      })}`;

      if (player.volume * 100 == 100)
        return await replyInteraction(client, message, reply_msg);

      player.setVolume(player.volume * 100 + 10);
      return await replyInteraction(client, message, reply_msg);
    } else if (id === "voldown") {
      if (!player) {
        collector.stop();
      }

      const reply_msg = `${client.i18n.get(language, "player", "voldown_msg", {
        volume: `${player.volume * 100 - 10}`,
      })}`;

      if (player.volume * 100 == 0)
        return await replyInteraction(client, message, reply_msg);

      player.setVolume(player.volume * 100 - 10);

      return await replyInteraction(client, message, reply_msg);
    } else if (id === "replay") {
      if (!player) {
        collector.stop();
      }
      await player["send"]({
        op: "seek",
        guildId: message.guild!.id,
        position: 0,
      });

      return await replyInteraction(
        client,
        message,
        `${client.i18n.get(language, "player", "replay_msg")}`
      );
    } else if (id === "queue") {
      if (!player) {
        collector.stop();
      }
      const song = player.queue.current;
      const qduration = `${formatduration(song!.length)}`;
      const thumbnail = `https://img.youtube.com/vi/${
        song!.identifier
      }/hqdefault.jpg`;

      let pagesNum = Math.ceil(player.queue.length / 10);
      if (pagesNum === 0) pagesNum = 1;

      const songStrings = [];
      for (let i = 0; i < player.queue.length; i++) {
        const song = player.queue[i];
        songStrings.push(
          `**${i + 1}.** [${song.title}](${song.uri}) \`[${formatduration(
            song.length
          )}]\`
            `
        );
      }

      const pages = [];
      for (let i = 0; i < pagesNum; i++) {
        const str = songStrings.slice(i * 10, i * 10 + 10).join("");

        const embed = new EmbedBuilder()
          .setAuthor({
            name: `${client.i18n.get(language, "player", "queue_author", {
              guild: message.guild!.name,
            })}`,
            iconURL: String(message.guild!.iconURL()),
          })
          .setColor(client.color)
          .setDescription(`**[${track.title}](${track.uri})**`)
          .addFields([
            {
              name: `${client.i18n.get(language, "player", "author_title")}`,
              value: `${song!.author}`,
              inline: true,
            },
            {
              name: `${client.i18n.get(language, "player", "request_title")}`,
              value: `${song!.requester}`,
              inline: true,
            },
            {
              name: `${client.i18n.get(language, "player", "duration_title")}`,
              value: `${formatduration(song!.length)}`,
              inline: true,
            },
            {
              name: `${client.i18n.get(language, "player", "download_title")}`,
              value: `**[${
                song!.title
              } - y2mate.com](https://www.y2mate.com/youtube/${
                song!.identifier
              })**`,
              inline: false,
            },
            {
              name: "list_song:",
              value: `${str == "" ? "  Nothing" : "\n" + str}`,
              inline: true,
            },
          ])
          .setColor(client.color)
          .setFooter({
            text: `${client.i18n.get(language, "player", "queue_footer", {
              page: `${i + 1}`,
              pages: `${pagesNum}`,
              queue_lang: `${player.queue.length}`,
              total_duration: qduration,
            })}`,
          });

        pages.push(embed);
      }
      message.reply({ embeds: [pages[0]], ephemeral: true });
    } else if (id === "clear") {
      if (!player) {
        collector.stop();
      }
      player.queue.clear();

      return await replyInteraction(
        client,
        message,
        `${client.i18n.get(language, "player", "clear_msg")}`
      );
    }
  });
};
