# InfoAPI
The standard for user config variables.

## FAQ for users
### How to add info into my config?
First, check what infos the plugin provides. For example, a chat formatting plugin might have an info called `player` which represents the player chatting, and an info called `message` which represents the message to be sent.

Then you can customize the chat message like this:
```
<${player}> ${message}
```

If the player is called `Steve` and the message is `Hello world`, the formatted chat message would become `<Steve> Hello world`.

Each `${...}` will be replaced by the info called `...`.

### How to add detailed info?
Different types of info have different details. For example, an info representing a player (we call it PlayerInfo) has a detail called `ip`. So if your config has the customized chat message like this:
```
<${player} @ ${player ip}> ${message}
```

This will become something like:
```
<Steve @ 12.34.56.78> Hello world
```

Simply add a space behind the info name, then add the detail name. You can repeat this process:

```
<${player} (${player health percent} health)> ${message}
```

A PlayerInfo has a detail RatioInfo called `health`, which represents the health of the player, so `${player health}` shows the player health. But we want a percentage. The RatioInfo has a detail StringInfo called `percent`, which converts the ratio into percentage, so we end up having this:

```
<Steve (95% health)> Hello world
```

### How to check what detailed info are available?
One way is to check for docs. Each plugin provides some details for different types of info. For example, a factions plugin might provide a detail for PlayerInfo called `faction`, which represents the player's faction. InfoAPI itself provides some common details, which you can find [here](builtin-info.md).

Another way is to use the `/info` command. `/info` displays the detail infos about a specific player, but you can click into them and see what they support.

### Escape codes
> - (Or: What if I really want to type `${}` without replacement?)
> - (Or: How to make multiple lines?)

The \\ character is reserved for "escaping" special symbols. Here are some conversions:
- If you really want to have a \\ character, type it twice: `\\`
- If you really want to have `${` without getting replaced, type `\${` (or `$\{` is also OK)
- If you want to insert a newline

**Warning**: In some cases, you may need to double each \\, e.g. instead of typing \\\\, you might need to type \\\\\\\\ due to YAML encoding problems.

## For developers:
### Providing info for InfoAPI
If your plugin stores data (esp. about players, etc.), you can expose your data to InfoAPI so that users can use these data in other plugins.

To provide info for InfoAPI, use the API methods on `InfoRegistry` in `onEnable`:
```php
public function onEnable(){
	// other normal stuff
	if(class_exists(InfoRegistry::class)){
		$registry = InfoRegistry::getInstance();
		$registry->addDetail(SomeInfo::class, "pluginname.modulename.infoname", function(SomeInfo $info){
			return new MyInfo(...);
		});
	}
}
```

`SomeInfo` is the type of the parent of the info you provide, and `MyInfo` is the type of the info provided.

The second parameter is a fully-qualified identifier for the info you provide. Users only need to type the part behind the last dot, but they can use the previous parts in case two infos have identical names (especially from different plugins). (For the first example above, `${infoname}`, `${modulename.infoname}` and `${pluginname.modulename.infoname}` will all be matched)

For example, if you provide a kills/deaths ratio for a player in a plugin called KDCounter, your code might look like this:

```php
public function onEnable(){
	// other normal stuff
	if(class_exists(InfoRegistry::class)){
		$registry = InfoRegistry::getInstance();
		$registry->addDetail(PlayerInfo::class, "kdcounter.kills", function(PlayerInfo $info){
			return new NumberInfo($this->getKills($info->getPlayer()));
		});
		$registry->addDetail(PlayerInfo::class, "kdcounter.deaths", function(PlayerInfo $info){
			return new NumberInfo($this->getDeaths($info->getPlayer()));
		});
		$registry->addDetail(PlayerInfo::class, "kdcounter.kd", function(PlayerInfo $info){
			return new RatioInfo($this->getKills($info->getPlayer()), $this->getDeaths($info->getPlayer()));
		});
	}
}
```

The second parameter is a closure that gets called if the event is matched. It should return an `Info` object that is the resolved result.

InfoAPI introduces some builtin subclasses for `Info`, but plugins may also implement new subclasses of such type. Please consult the API documentation for usage.

### Using InfoAPI for config
If your plugin uses placeholders in your config, you can use InfoAPI so that users can use data from other plugins.

To use InfoAPI, add `InfoAPI` to the `depend` attribute in plugin.yml 
```yaml
depend: [InfoAPI]
```

In the class where you use the InfoAPI value, call this method (all classes are under the namespace `SOFe\InfoAPI`):
```php
$result = InfoAPI::resolveTemplate($configValue, [
	"player" => new PlayerInfo($player),
	"message" => new StringInfo($message),
	// etc.
]);
```

`$configValue` is the template string supplied by the user. The returned result is the formatted string. An InvalidArgumentException is thrown if the info cannot be fully resolved.
