# Resumen del Plugin: PuentesFalion

Este documento resume todas las clases, funcionalidades y conceptos implementados durante la creación del plugin PuentesFalion para Minecraft.

---

## 1. **Clases Implementadas**

### 1.1. `GameFalion.java`
Clase principal del plugin que inicializa las funciones principales:
- Registra los equipos.
- Maneja el comando `/team`.
- Configura los mensajes personalizados a través del archivo `messages.yml`.

```java
package me.main.gameFalion;

import me.main.gameFalion.Equipos.Equipos;
import me.main.gameFalion.Equipos.TeamCommandListener;
import org.bukkit.plugin.java.JavaPlugin;

public class GameFalion extends JavaPlugin {
    private Equipos equipos;
    private MessageManager messageManager;

    @Override
    public void onEnable() {
        getLogger().info("El plugin PuentesFalion ha sido habilitado.");

        // Inicializar el gestor de mensajes
        messageManager = new MessageManager(this);
        messageManager.saveDefaultMessages();

        // Inicializar equipos
        equipos = new Equipos(messageManager);

        // Registrar el listener y el comando
        getServer().getPluginManager().registerEvents(new TeamCommandListener(equipos, messageManager), this);
        this.getCommand("team").setExecutor(new TeamCommandListener(equipos, messageManager));
    }

    @Override
    public void onDisable() {
        getLogger().info("El plugin PuentesFalion ha sido deshabilitado.");
    }

    public MessageManager getMessageManager() {
        return messageManager;
    }
}
```

---

### 1.2. `MessageManager.java`
Clase encargada de manejar los mensajes configurables desde `messages.yml`.

```java
package me.main.gameFalion;

import org.bukkit.ChatColor;
import org.bukkit.configuration.file.FileConfiguration;
import org.bukkit.configuration.file.YamlConfiguration;
import org.bukkit.plugin.java.JavaPlugin;

import java.io.File;
import java.io.IOException;

public class MessageManager {
    private final JavaPlugin plugin;
    private FileConfiguration config;
    private File file;

    public MessageManager(JavaPlugin plugin) {
        this.plugin = plugin;
        createMessagesFile();
    }

    private void createMessagesFile() {
        file = new File(plugin.getDataFolder(), "messages.yml");
        if (!file.exists()) {
            file.getParentFile().mkdirs();
            plugin.saveResource("messages.yml", false);
        }
        config = YamlConfiguration.loadConfiguration(file);
    }

    public void saveDefaultMessages() {
        if (!config.contains("team-join-success")) {
            config.set("team-join-success", "&aTe has unido al equipo {team}.");
        }
        if (!config.contains("team-join-failure")) {
            config.set("team-join-failure", "&cNo se pudo unir al equipo. Puede que ya estés en un equipo o el equipo no exista.");
        }
        if (!config.contains("only-players")) {
            config.set("only-players", "&cEste comando solo puede ser ejecutado por jugadores.");
        }
        if (!config.contains("usage-team-command")) {
            config.set("usage-team-command", "&cUso incorrecto. Usa /team <Azul|Verde|Rojo|Amarillo>");
        }
        if (!config.contains("broadcast-team-join")) {
            config.set("broadcast-team-join", "&e{player} se ha unido al equipo {team}.");
        }

        try {
            config.save(file);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public String getMessage(String key) {
        if (config.contains(key)) {
            return ChatColor.translateAlternateColorCodes('&', config.getString(key));
        }
        return ChatColor.RED + "Mensaje no encontrado: " + key;
    }
}
```

---

### 1.3. `Equipos.java`
Clase que gestiona la lógica de los equipos, como unir jugadores a equipos y verificar si están en el mismo equipo.

```java
package me.main.gameFalion.Equipos;

import me.main.gameFalion.MessageManager;
import org.bukkit.entity.Player;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Set;
import java.util.UUID;

public class Equipos {
    private final HashMap<String, Set<UUID>> equipos;
    private final Tag tagManager;
    private final MessageManager messageManager;

    public Equipos(MessageManager messageManager) {
        equipos = new HashMap<>();
        equipos.put("Azul", new HashSet<>());
        equipos.put("Verde", new HashSet<>());
        equipos.put("Rojo", new HashSet<>());
        equipos.put("Amarillo", new HashSet<>());

        tagManager = new Tag();
        this.messageManager = messageManager;
    }

    public boolean unirseEquipo(String equipo, Player player) {
        UUID playerUUID = player.getUniqueId();
        if (equipos.containsKey(equipo)) {
            for (Set<UUID> miembros : equipos.values()) {
                if (miembros.contains(playerUUID)) {
                    return false;
                }
            }

            equipos.get(equipo).add(playerUUID);
            tagManager.asignarJugadorAEquipo(player, equipo);
            return true;
        }
        return false;
    }
}
```

---

### 1.4. `TeamCommandListener.java`
Clase que maneja el comando `/team`.

```java
package me.main.gameFalion.Equipos;

import me.main.gameFalion.MessageManager;
import org.bukkit.Bukkit;
import org.bukkit.command.Command;
import org.bukkit.command.CommandExecutor;
import org.bukkit.command.CommandSender;
import org.bukkit.entity.Player;
import org.bukkit.event.Listener;

public class TeamCommandListener implements CommandExecutor, Listener {
    private final Equipos equipos;
    private final MessageManager messageManager;

    public TeamCommandListener(Equipos equipos, MessageManager messageManager) {
        this.equipos = equipos;
        this.messageManager = messageManager;
    }

    @Override
    public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        if (!(sender instanceof Player)) {
            sender.sendMessage(messageManager.getMessage("only-players"));
            return true;
        }

        Player player = (Player) sender;

        if (args.length != 1) {
            player.sendMessage(messageManager.getMessage("usage-team-command"));
            return true;
        }

        String equipo = args[0];
        boolean unido = equipos.unirseEquipo(equipo, player);

        if (unido) {
            player.sendMessage(messageManager.getMessage("team-join-success").replace("{team}", equipo));
            Bukkit.broadcastMessage(messageManager.getMessage("broadcast-team-join")
                    .replace("{player}", player.getName())
                    .replace("{team}", equipo));
        } else {
            player.sendMessage(messageManager.getMessage("team-join-failure"));
        }
        return true;
    }
}
```

---

### 1.5. `Tag.java`
Clase que asigna colores y prefijos a los jugadores según su equipo.

```java
package me.main.gameFalion.Equipos;

import org.bukkit.Bukkit;
import org.bukkit.ChatColor;
import org.bukkit.entity.Player;
import org.bukkit.scoreboard.Scoreboard;
import org.bukkit.scoreboard.ScoreboardManager;
import org.bukkit.scoreboard.Team;

public class Tag {
    private final Scoreboard scoreboard;

    public Tag() {
        ScoreboardManager manager = Bukkit.getScoreboardManager();
        this.scoreboard = manager.getNewScoreboard();

        for (String teamName : new String[]{"Azul", "Verde", "Rojo", "Amarillo"}) {
            Team team = scoreboard.registerNewTeam(teamName);

            switch (teamName) {
                case "Azul":
                    team.setColor(ChatColor.BLUE);
                    team.setPrefix(ChatColor.BLUE + "[Azul] ");
                    break;
                case "Verde":
                    team.setColor(ChatColor.GREEN);
                    team.setPrefix(ChatColor.GREEN + "[Verde] ");
                    break;
                case "Rojo":
                    team.setColor(ChatColor.RED);
                    team.setPrefix(ChatColor.RED + "[Rojo] ");
                    break;
                case "Amarillo":
                    team.setColor(ChatColor.YELLOW);
                    team.setPrefix(ChatColor.YELLOW + "[Amarillo] ");
                    break;
            }
        }
    }

    public void asignarJugadorAEquipo(Player player, String equipo) {
        Team team = scoreboard.getTeam(equipo);
        if (team != null) {
            for (Team t : scoreboard.getTeams()) {
                t.removeEntry(player.getName());
            }

            team.addEntry(player.getName());
            player.setScoreboard(scoreboard);
        }
    }
}
```

---

## 2. **Archivo `messages.yml`**
Archivo de configuración para mensajes personalizables.

```yaml
team-join-success: "&aTe has unido al equipo {team}."
team-join-failure: "&cNo se pudo unir al equipo. Puede que ya estés en un equipo o el equipo no exista."
only-players: "&cEste comando solo puede ser ejecutado por jugadores."
usage-team-command: "&cUso incorrecto. Usa /team <Azul|Verde|Rojo|Amarillo>"
broadcast-team-join: "&e{player} se ha unido al equipo {team}."
```

---

### Notas Finales

- **Estructura del Proyecto**: Asegúrate de organizar las clases correctamente en paquetes.
- **Mensajes Personalizables**: Edita `messages.yml` para cambiar los mensajes que vean los jugadores.
- **Colores de Equipos**: Los colores y prefijos se aplican automáticamente en el chat y el tab según el equipo.

Si tienes dudas o necesitas más ayuda, no dudes en pedírmelo. 😊