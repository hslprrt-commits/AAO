package me.companion;

import org.bukkit.Location;
import org.bukkit.command.Command;
import org.bukkit.command.CommandSender;
import org.bukkit.entity.Entity;
import org.bukkit.entity.Monster;
import org.bukkit.entity.Player;
import org.bukkit.entity.Villager;
import org.bukkit.plugin.java.JavaPlugin;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

public class CompanionPlugin extends JavaPlugin {

    private static final double FOLLOW_DISTANCE = 4.0;   // يبدأ يلحقك إذا ابتعدت أكثر
    private static final double TELEPORT_DISTANCE = 25.0; // يتنقل لك إذا بعدت مرة
    private static final double SCAN_RADIUS = 8.0;        // مدى رؤية الوحوش
    private static final double ATTACK_RANGE = 2.5;
    private static final double ATTACK_DAMAGE = 4.0;
    private static final long ATTACK_COOLDOWN_MS = 1000;

    // owner UUID -> bot
    private final Map<UUID, Villager> bots = new HashMap<>();
    private final Map<UUID, Long> lastAttack = new HashMap<>();
    private final Set<UUID> staying = new HashSet<>();   // أصحاب بوتات واقفة
    private final Set<UUID> noAttack = new HashSet<>();  // أصحاب بوتات الهجوم مقفل عندهم

    private static final int MAX_NAME_LENGTH = 32;

    @Override
    public void onEnable() {
        getServer().getScheduler().runTaskTimer(this, this::tickBots, 10L, 10L);
    }

    @Override
    public void onDisable() {
        bots.values().forEach(Entity::remove);
        bots.clear();
    }

    @Override
    public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
        if (!(sender instanceof Player player)) {
            sender.sendMessage("Players only.");
            return true;
        }
        String usage = "/bot spawn | remove | follow | stay | attack on|off | name <name>";
        if (args.length == 0) {
            player.sendMessage(usage);
            return true;
        }
        UUID id = player.getUniqueId();
        String sub = args[0].toLowerCase();
        boolean needsBot = sub.equals("follow") || sub.equals("stay")
                || sub.equals("attack") || sub.equals("name");
        Villager current = bots.get(id);
        if (needsBot && (current == null || !current.isValid())) {
            player.sendMessage("You have no bot. Use /bot spawn first.");
            return true;
        }
        switch (sub) {
            case "follow" -> {
                staying.remove(id);
                player.sendMessage("Bot will follow you.");
            }
            case "stay" -> {
                staying.add(id);
                player.sendMessage("Bot will stay here.");
            }
            case "attack" -> {
                if (args.length < 2 || !(args[1].equalsIgnoreCase("on") || args[1].equalsIgnoreCase("off"))) {
                    player.sendMessage("/bot attack on|off");
                    return true;
                }
                if (args[1].equalsIgnoreCase("on")) noAttack.remove(id); else noAttack.add(id);
                player.sendMessage("Bot attack: " + args[1].toLowerCase());
            }
            case "name" -> {
                if (args.length < 2) {
                    player.sendMessage("/bot name <name>");
                    return true;
                }
                String name = String.join(" ", java.util.Arrays.copyOfRange(args, 1, args.length)).trim();
                if (name.length() > MAX_NAME_LENGTH) name = name.substring(0, MAX_NAME_LENGTH);
                current.setCustomName(name);
                player.sendMessage("Bot renamed.");
            }
            case "spawn" -> {
                removeBot(player.getUniqueId());
                Villager v = player.getWorld().spawn(player.getLocation(), Villager.class, bot -> {
                    bot.setCustomName(player.getName() + "'s Bot");
                    bot.setCustomNameVisible(true);
                    bot.setInvulnerable(true);
                    bot.setPersistent(false); // ما ينحفظ مع السيرفر
                });
                bots.put(player.getUniqueId(), v);
                player.sendMessage("Bot spawned.");
            }
            case "remove" -> {
                removeBot(player.getUniqueId());
                player.sendMessage("Bot removed.");
            }
            default -> player.sendMessage(usage);
        }
        return true;
    }

    private void removeBot(UUID owner) {
        Villager old = bots.remove(owner);
        if (old != null) old.remove();
        lastAttack.remove(owner);
        staying.remove(owner);
        noAttack.remove(owner);
    }

    /** "الذكاء": آلة حالات بسيطة — هجوم > لحاق > وقوف */
    private void tickBots() {
        Iterator<Map.Entry<UUID, Villager>> it = bots.entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry<UUID, Villager> e = it.next();
            Villager bot = e.getValue();
            Player owner = getServer().getPlayer(e.getKey());

            if (owner == null || !owner.isOnline() || !bot.isValid()) {
                bot.remove();
                lastAttack.remove(e.getKey());
                staying.remove(e.getKey());
                noAttack.remove(e.getKey());
                it.remove();
                continue;
            }
            if (!bot.getWorld().equals(owner.getWorld())) {
                bot.teleport(owner.getLocation());
                continue;
            }

            Monster target = noAttack.contains(e.getKey()) ? null : nearestMonster(bot);
            if (target != null) {
                fight(e.getKey(), bot, target);
            } else if (!staying.contains(e.getKey())) {
                follow(bot, owner);
            }
        }
    }

    private Monster nearestMonster(Villager bot) {
        Monster best = null;
        double bestDist = Double.MAX_VALUE;
        for (Entity en : bot.getNearbyEntities(SCAN_RADIUS, SCAN_RADIUS / 2, SCAN_RADIUS)) {
            if (en instanceof Monster m && m.isValid()) {
                double d = m.getLocation().distanceSquared(bot.getLocation());
                if (d < bestDist) { bestDist = d; best = m; }
            }
        }
        return best;
    }

    private void fight(UUID owner, Villager bot, Monster target) {
        double dist = target.getLocation().distance(bot.getLocation());
        if (dist <= ATTACK_RANGE) {
            long now = System.currentTimeMillis();
            if (now - lastAttack.getOrDefault(owner, 0L) >= ATTACK_COOLDOWN_MS) {
                bot.swingMainHand();
                target.damage(ATTACK_DAMAGE, bot);
                lastAttack.put(owner, now);
            }
        } else {
            bot.getPathfinder().moveTo(target.getLocation(), 1.3);
        }
    }

    private void follow(Villager bot, Player owner) {
        Location ownerLoc = owner.getLocation();
        double dist = ownerLoc.distance(bot.getLocation());
        if (dist > TELEPORT_DISTANCE) {
            bot.teleport(ownerLoc);
        } else if (dist > FOLLOW_DISTANCE) {
            bot.getPathfinder().moveTo(ownerLoc, 1.2);
        }
    }
}
