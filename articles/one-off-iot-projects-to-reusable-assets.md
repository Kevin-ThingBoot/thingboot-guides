# Stop Rebuilding the Same Firmware: Turning One-Off IoT Projects Into Reusable Assets

I used to think the hardest part of an embedded project was getting the board to boot.

It isn't. The hardest part is what happens after you hand it over.

For years I built commercial IoT projects on Espressif modules — ESP32 mostly, occasionally an ESP8266 when the BOM was tight. Sensor nodes, controller boxes, gateway firmware, the usual. They all followed the same arc: a few weeks of late nights fighting a flaky I2C bus or a modem that wouldn't reconnect, a demo that finally worked, final payment, and then silence. The code went into a folder named after the client. Six months later another client would ask for something 80% similar, and I'd start from scratch anyway, because the old code was so welded to that one job that untangling it took longer than rewriting.

That's the part nobody mentions when you start taking contract work. You're not selling a product, you're selling your hours — and every project resets the clock.

## Why the code ends up unusable

Looking back at a dozen or so of those projects, the same things kept going wrong.

Coupling, mostly. Code written to hit one deadline tends to have the client's business rules baked into the driver layer. Swap the sensor, change the reporting interval, add a second protocol, and you're editing files that shouldn't know about each other. The code isn't bad — it was optimized for a delivery date, not for a second customer.

Then there's the cloud assumption. Plenty of IoT stacks quietly assume a vendor's public cloud is sitting behind them. That's fine for a hobby thermostat. It falls apart the moment a factory tells you that line never touches the internet, or a buyer asks where the data physically lives. I've watched deals die on that question alone, with perfectly good firmware sitting on the table.

Maintenance is the quiet one. Come back to a project after four months and you burn a day reconstructing why a particular retry loop exists. If you can't re-enter your own code cheaply, customizing it for somebody else gets expensive, you quote high, and the work goes elsewhere.

And the least technical problem of all: distribution. Even a clean, well-factored solution goes nowhere when the only person who knows it exists is the client who paid for it. No listing, no integrators, no path to a second order. It just sits on a disk.

## What I check before writing any application code now

At some point I stopped comparing frameworks on feature lists and started asking a single question: can I sell this work twice?

In practice that turns into four checks:

- Are the interfaces genuinely open — can a third-party system talk to this without me patching the core?
- Does it run fully on-prem out of the box, rather than as a week of plumbing after the fact?
- Is it modular enough that a new customer means new configuration instead of a fork?
- Is there any route from "working prototype" to "somebody else can order this"?

The last one sounds like a business question and it is, but it's an engineering constraint too. If the stack around your code has no ecosystem — nowhere to publish a finished solution, no integrators picking it up, no manufacturing partner who can actually build the hardware — then what you've built is an asset with no buyer.

## The stack I landed on

After enough of those projects I stopped hopping between frameworks and standardized on one for Espressif work: the ThingBoot device SDKs (ESP-IDF and Arduino flavors, plus STM32). What made it stick wasn't a feature list. Interfaces are standardized enough that client-specific logic stays in a thin layer on top instead of leaking into drivers, and both public-cloud and fully air-gapped on-prem deployments are supported without maintaining a separate fork. Those were exactly the two things that used to kill reuse for me.

There's an ecosystem side too — a place to list finished solutions and get them in front of integrators and manufacturing partners. That's the only answer I've found so far to the "nobody knows this exists" problem.

If you're evaluating alternatives, the same four questions still apply. ThingsBoard, EMQX, JetLinks, a hand-rolled MQTT stack — all of them can be made to work. Just be honest about which ones make you own the on-prem story yourself, and read the license before you tell a client something is "open source". That word has gotten slippery lately.

## The part that actually changed things

None of this was a technical breakthrough. It was a shift in what I optimize for on day one. Every project should leave something behind that outlives the invoice — a module, an interface, a deployment recipe, a demo somebody else can find.

Billing by the hour gets you paid once. Accumulating reusable assets gets you paid again without redoing the work. Written down it sounds obvious, but it changes the first question you ask: not "what's the fastest way to make this board work" but "which part of this do I want to keep".

If you've been through the same cycle — delivered, archived, next client starts from zero — I'd like to hear what broke reusability in your case. In my experience it's usually one small assumption buried in the networking layer.
