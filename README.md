![Logo](./Resources/Logo.png)

# MrConnectome

An API to access a crude recreation of the *c. Elegans* connectome for mid-level programs.


*Note: I am an engineer, not a neuroscientist, so this stuff is probably horribly wrong.
Either way, I will go at it with oblivious vigor.*

## Demo
I stuck this in Cave Story to see if it worked. Long story short, it did:
https://github.com/DrGlaucous/CaveS-Public/tree/main/Connectome

Also, this is where I actually have the library, currently written in Rust. I need to bring it over here sometime.

## CSV format:

| Neuron | Persistence | Threshold | Output 1 | Weight 1 | Output 2 | Weight 2 | ... |
| - | - | - | - | - | - | - | - |
| ADAL | 10 | 30 | ADAR | 1 | AIBR | 2 |
| MDL04 | -1 | 3000 |

- `Neuron` - Name of the neuron this entry will affect
- `Persistence` - How many `tick()` cycles can happen between weight adjustments before the neuron's weight will be reset. Values less than 0 are not reset.
- `Threshold` - The value that the neuron's weight must exceed before it will fire.
- `Output X` - Name of a neuron that `Neuron` will stimulate when it fires
- `Weight X` - The value that the other neuron's weight will be changed by when stimulated by this one

### Notes
- There can be as many `output` + `weight` pairs as desired, new pairs can simply be added at the end of the line. A neuron with no outputs (for instance, `MDL04`) are typically used as an interface to outside variables. In the example used here, `MDL04` isn't a real neuron in the connectome, but represents what would be a muscle in the real organism. Because this has no outputs, the threshold value doesn't have as much importance here, so long as its set to something reasonably large so that the weight isn't reset before it gets a chance to be read.


## Resources + credits

Original connectome author: Timothy Busbice, (c) 2014

Original source:
- http://www.connectomeengine.com/

Other useful articles and repos:
- https://github.com/openworm/openworm_docs/blob/master/docs/Resources/resources.md
- https://www.wormatlas.org/neuronalwiring.html



## Notes for me
<details>

<summary>Notes</summary>


## Imported Theory of operation

Currently, the simulated connectome runs on python.
There is an integer dictionary *(hash map)* with an entry for each neuron, and output point, containing the neuron's total accumulated weight from that cycle.

There is a single "threshold" shared by all neurons, and when a neuron's entry in the hash map is exceeded, the neuron fires and runs a corresponding function that then either stimulates or inhibits other neurons with various weighting. After this function is ran, it resets that neuron's stimulation factor to 0.

*I now realize that charges will accumulate on the neuron until it fires and resets, and it does not require a simultaneous multi-fire from other neurons to trigger it.*


There are several "Stimulation Points" in the main function that forcibly fire a selected group of neurons to set a behavior in the larger connectome.

For instance, stimulating "Food" fires:
```
dendriteAccumulate("ADFL")
dendriteAccumulate("ADFR")
dendriteAccumulate("ASGR")
dendriteAccumulate("ASGL")
dendriteAccumulate("ASIL")
dendriteAccumulate("ASIR")
dendriteAccumulate("ASJR")
dendriteAccumulate("ASJL")
```
and makes the connectome think there's food right ahead


There are several output points, which take weight inputs like neurons, but don't have a "firing" function. These are for external interfaces like muscles.

The weights on the muscles correspond to the speed of the output motor. All "Left" motor neurons are summed in a single group, and all "Right" motor neurons are summed in a single group.

The interface code currently sums the absolute value of all motors (left and right) and then clamps them between 75 and 150.

there are several cases for movement:
- `Left` and `Right` are both 0: halt motors
- `Left` and `Right` are both less than 0: max speed for both motors, extra left or right bias depending on the ratio of right/left.
- `Left` is greater than 0 and `Right` is less than 0: Spin right motor (turn left)
- `Left` is less than 0 and `Right` is greater than 0: Spin left motor (turn right)
- `Left` and `Right` are larger than 0: spin motors forward at summed speed, with extra bias in the ratio of right/left.

then reset the `Left` and `Right` accumulation sums (which are global for some reason)


Motor weights are not zeroed, and will continue to increment or decrement as-needed (unless clamped between values)

### Notes:
There is only one hash map for the neuron stim values, so neuron behavior is influenced by update order, which is an iterator through a hash map.

## Optimized theory of operation

This is what *I* want to try making

There will be two "neuron weight" variables per neuron, one for reference, and one for next update. This way, all the neurons can update off current values, then the weight will be swapped for the next update.

I'll use a hash map to store each neuron weight like before, but also, I'll store the keys of all the neurons this neuron will update, along with their update weights.

I want to make it so we can read in new neuron connections and weights from a csv file, instead of a long, long list of hard-coded functions (as is currently done).

```
//something kind of like this... (psuedo-cpp)
class Neuron {

    public:

    Neuron() {

    }

    int get_weight() {

    }

    //set the value of the next weight
    void increment_weight(int weight) {
        weights[next_weight_index] += weight;
    }

    void process_neuron(HashMap& hash_map) {

        if(weights[weight_index] > threshold) {
            for(auto key: output_connections) {
                //todo: look at same hash-map from in here
                hash_map.get(key.key).increment_weight(key.weight)
            }
        }
    }

    //needs to be called from outside after all updates have been made to all neurons
    void flip_weight_ref() {
        if(weight_index == 0) {
            weight_index = 1;
            next_weight_index = 0;
        } else {
            weight_index = 0;
            next_weight_index = 1;
        }
    }


    private:
    
    int threshold = 30; //current hard-coded value for now

    //current and next weight, 
    int weights[2] = {};
    size_t weight_index = 0;
    size_t next_weight_index = 1;

    //contains the weight associated with this key
    struct key_weight {
        std::string key;
        int weight;
    }

    std::vector<key_weight> output_connections;





}
```

</details>
