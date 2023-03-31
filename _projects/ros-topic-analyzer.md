---
layout: page
title: ROS Topic Analyzer
description: TUI for live topic monitoring in ROS systems
importance: 2
img: assets/img/ros_topic_analyzer/project_image.png
category: work
---

## Overview

ROS is an amazing piece of software for robtics development, but
in a system like the AVC vehicle, it is sometimes hard to keep track
of every single topic that we are wanting to monitor. Running `rostopic hz \topic`
every time we want to double check a publishing rate, then `rostopic info \topic`,
then finally `rostopic echo \topic` becomes quite tedious when you have a system
with 6 radars, 2 lidars, 6 cameras, and multiple other measurements coming
through at any given time.

To fix this issue, and facilitate system monitoring at scale, I created
a TUI to monitor specific aspects of our ROS system, called the [ROS Topic Analyzer](https://github.com/dominicfmazza/ROS-Topic-Analyzer)
My requirements for the system were:

- User configuration via a JSON schema
- 3 separate displays in the UI
  - Config view that displays JSON validation feedback
  - List view with all topics and rate information
  - Focus view with more detailed topic information,
    including publishers, subscribers, and message data
- Keybinds and mouse input over SSH

In my development, I chose to use `asciimatics` as my TUI library of choice,
`sqlite3` for management of topic information, and of course used the
`rospy` and `rostopic` APIs for querying information from master.

## Implementation

### JSON Schema

At the root of the system is configuring a JSON file for the topics we want to monitor.
The JSON schema I designed is quite simple:

```javascript
{
  "topics": [
    {
      "display_name": "Name", //a plaintext name for the topic
      "handle": "/namespace/namespace/topic", //the actual topic address
      "hz_target": "100.0" //a float for a target hz rate
    }
  ]
}
```

All that a user needs to specify is a plaintext name for the topic for readability,
the topic itself, and the target rate for the topic, and the ROS backend will handle
the rest.

### Config Selection

The next part of the system the user encounters is the topic selection screen.

<div class="container">
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/ros_topic_analyzer/project_image.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
  <div class="caption">
    Config Selection 
  </div>
</div>

The topic selection screen shows all the configuration files placed in the `cfg/`
directory in the package. When you attempt to select a configuration file, it will
automatically:

- Validate structure of the JSON file
- Validate the topic handle and whether or not it is published
- Validate the target rate given

If there are issues with any of the above, you will see an exception printed
in the `JSON Configuration Errors` box.

For example, if you attempt to choose
the `../` directory:

<div class="container">
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/ros_topic_analyzer/invalid_file.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
  <div class="caption">
     Invalid File Selection 
  </div>
</div>

Another example, if one of the topics is not published yet:

<div class="container">
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/ros_topic_analyzer/not_published.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
  <div class="caption">
     Unpublished Topic 
  </div>
</div>

### ROS Backend and Database

In my initial development, I noticed that attempting to get the ROS thread
and a TUI library to play nice with each other in a single thread was a fools errand.
To work around this, I spawned a separate thread for the ROS node handling queries to the
and entering database information. This ROS node handled the requesting of all information
from the ROS system for both list views and focus views.

The way we get information from ROS is simple:

1. Spawn a thread to run the ROS node and pass in the config and the database model:

   ```python
   def run_node(config, model):
       node = Node(config, model)
       try:
           node.run()
       except rospy.ROSInterruptException:
           sys.exit(0)
   ```

2. Parse the JSON, creating a database entry for each topic and storing
   `ROSTopicHz` objects and rate subscribers:

   ```python
   # get the message type for the topic
   (msg_object, _, _) = rostopic.get_topic_class(topic["handle"])

   # creates a ROSTopicHZ class to monitor topic HZ for
   # the last 100 messages
   monitor = rostopic.ROSTopicHz(100)

   # subscribes to topic to monitor hz
   hz_subscriber = rospy.Subscriber(
       topic["handle"],
       msg_object,
       monitor.callback_hz,
   )

   # create a dictionary to be added to database and adds
   topic = {
       "title": topic["display_name"],
       "handle": topic["handle"],
       "rate": "0.0",
       "target": topic["hz_target"],
       "type": msg_object._type,
   }
   self._model.add(topic)

   # add monitor to the list
   self.monitors.append(monitor)

   # add subscriber to the list
   self.hz_subscribers.append(hz_subscriber)
   ```

3. Create a timer and callback to automatically refresh the topic information
   at a fixed rate of 4 hz. This timer callback determines whether or not we
   are looking at the list view or the focus view based on the database wrapper.
   If the view switches to a focus view, it sets `current_id` to the index of
   the topic we are interested in. If we go back to list view, then `current_id = None`.

   ```python
   # callback to update database values 4 times a second
   def timer_callback(self, timer):

       # if there is no currently selected topic,
       # just update the hz for every topic
       if self._model.current_id is None:
           for index in range(self.length):
               hz = self.get_hz(index)
               self._model.update_topic_hz(f"{hz:.2f}", index + 1)

       # if there is a currently selected topic, get its message and its
       # publishers/subscribers as well as only updating the topic's hz
       if self._model.current_id is not None:
           message = ""
           publishers = ""
           subscribers = ""
           try:
               # get hz for topic
               hz = self.get_hz(self._model.current_id - 1)

               # update the hz
               self._model.update_topic_hz(f"{hz:.2f}", self._model.current_id - 1)

               # gets current topic from model
               topic = self._model.get_current_topic()

               # waits for a singular message
               message = rospy.wait_for_message(
                   topic[2],
                   self.hz_subscribers[self._model.current_id - 1].data_class,
                   timeout=5,
               )

               # makes sure the messages wrap correctly
               message = "\n".join(
                   [
                       "\n".join(
                           textwrap.wrap(
                               line,
                               70,
                               break_long_words=False,
                               replace_whitespace=False,
                           )
                       )
                       for line in str(message).splitlines()
                       if line.strip() != ""
                   ]
               )

               # gets pub and subs for topic
               publishers, subscribers = self.get_pubsubs(topic[2])

           # catch errors related to topic not being up
           except Exception:
               message = "TOPIC NOT ONLINE\n"
               publishers = "TOPIC NOT ONLINE\n"
               subscribers = "TOPIC NOT ONLINE\n"

           # update database
           self._model.update_topic_info(
               message,
               publishers,
               subscribers,
           )
   ```

This database is operated in a thread safe manner as the ROS node thread
is the only thread that is modifying the database at any given time, and
the TUI thread is responsible for updating the `current_id` of the database
wrapper.

In terms of database implementation, its a simple wrapper for `sqlite3` queries.

## List View

The list view is the view you will encounter after selecting a valid configuration.
The view looks like this:

<div class="container">
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/ros_topic_analyzer/list_view.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
  <div class="caption">
     List View 
  </div>
</div>

The view shows the following pieces of information:

- `TITLE`: A plaintext name for the topic
- `HANDLE`: The handle for the topic
- `RATE (hz)`: The current rate for the topic
- `TARGET (hz)`: The target rate for the topic
- `TYPE`: The ROS message type of the topic

This view is updated at 4 hz. For more implementation details, view the [source](https://github.com/dominicfmazza/ROS-Topic-Analyzer/blob/main/src/display.py)

## Focus View

The focus view is shown when you select a topic from the list view. You can see
all of the information displayed in the list view, as well as the topic's
publishers, subscribers, and message contents.

<div class="container">
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/ros_topic_analyzer/focus_view.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
  <div class="caption">
     List View 
  </div>
</div>

This view is updated at 2 hz. For more implementation details, view the [source](https://github.com/dominicfmazza/ROS-Topic-Analyzer/blob/main/src/display.py)
Additionally, with topics containing a large amount information, such as image topics,
pointclouds, and radar objects, the data will take longer to display initially due to
reformatting the message into a readable dilineated string.

## Final Thoughts

This project was a ton of fun! As someone who mostly develops in C++ or Python in the backend,
it was a unique challenge to build a tool that is meant to be more user facing. The process of using
`asciimatics` and `sqlite3` was honestly a breeze, and I definitely think any other terminal tools
I develop in the future will make use of them. If you have any questions, comments, or improvements,
definitely submit an issue on [github](https://github.com/dominicfmazza/ROS-Topic-Analyzer/issues). Thanks for
stopping by!
