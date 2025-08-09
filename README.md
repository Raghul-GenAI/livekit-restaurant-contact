# livekit-restaurant-contact

https://medium.com/@brianmwangi_dev/voice-assistant-agent-livekit-firebase-and-freeswitch-efaedf34757e

Over the past couple of months there has been an increase in the number of startups building Voice Assistants, but the one problem they all share is how to get the conversation better and as close as possible to a call center human responder. While coding agents can now handling building these systems there’s still a gap in information on the flow of these systems: SIP (Session Initiation Protocol). SIP is essential for allowing agents to initiate, manage and terminate the voice and video calls over the internet which have been managed by VoIP like Twilio and Telynx.

For example imagine a restaurant business, it has overwhelming reservation calls, while still having to manage orders. An AI agent can really be of help here. The restaurant would use a VoIP but how do they ensure they have the right data for the right customer, how do they initiate the conversation, ensure the communication is right? and be able to end the call once the data has been captured. So we need SIP for that and to be able to connect to traditional network that’s where SIP trunking comes in.

Press enter or click to view image in full size

Let’s dive in on how to build the restaurant agent, We will use FreeSwitch to handle the calls, Firebase to store the data and LiveKit Agents to manage the conversation.

Setting Up
To ensure we keep this simple and straight forward, the assumption is you have Firebase account set up, create a new project. We will only use Firestore for now. But you need to head on over to the project settings and service account tab and generate a new private key.

Press enter or click to view image in full size

You also need to set up a python project with virtual environment. Now install the following dependencies.

pip install firebase-admin "livekit-agents[deepgram,openai,cartesia,silero,turn-detector]~=1.0" \
  "livekit-plugins-noise-cancellation~=0.2" \
  "python-dotenv"
Once complete, we need to set up the required keys, needed to communicate with OpenAI apis and Livekit. You can get some tokens to use to test with before you add your pay for any of the services, Deepgram is a good option. Create a .env and add the following, you can omit Cartesia or retain it.

DEEPGRAM_API_KEY=<Your Deepgram API Key>
OPENAI_API_KEY=<Your OpenAI API Key>
CARTESIA_API_KEY=<Your Cartesia API Key>
LIVEKIT_API_KEY=<your API Key>
LIVEKIT_API_SECRET=<your API Secret>
LIVEKIT_URL=<your LiveKit URL>
GROQ_API_KEY=<your groq api Secret>
We will be using Docker to run FreeSwitch and Livekit server all together locally, so create a Dockerfile and docker-compose.yml file to create the config and run the app. Here’s how it should look like

We define the Dockerfile

# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Expose port for FreeSWITCH socket connection
EXPOSE 8000

CMD ["python", "main.py"]
and compose, reference the Dockerfile in the backend config using . since its located in the same directory. We will talk about Freeswitch later on, and how to set it up for testing using its referenced config.

version: '3'
services:
  freeswitch:
    image: drachtio/freeswitch:latest
    ports:
      - "5060:5060/udp"    # SIP signaling
      - "5080:5080/udp"    # SIP signaling (alternative)
      - "8021:8021"        # Event Socket for LiveKit
      - "16384-16394:16384-16394/udp"  # RTP media ports
    volumes:
      - ./freeswitch-config:/etc/freeswitch
    environment:
      - FREESWITCH_LOG_LEVEL=INFO
    networks:
      - restaurant-net
    depends_on:
      - livekit
  
  # LiveKit server for media processing
  livekit:
    image: livekit/livekit-server:latest
    ports:
      - "7880:7880"   # WebSocket
      - "7881:7881"   # WebSocket with TLS  
      - "7882:7882/udp"  # TURN/UDP
      - "50000-60000:50000-60000/udp"  # LiveKit media ports
    environment:
      - LIVEKIT_KEYS=devkey:secret
      - LIVEKIT_CONFIG=/etc/livekit.yaml
    volumes:
      - ./livekit-config.yaml:/etc/livekit.yaml
    networks:
      - restaurant-net

  # Restaurant AI Assistant
  backend:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - .:/app
      - ./service.json:/app/service.json
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - DEEPGRAM_API_KEY=${DEEPGRAM_API_KEY}
      - CARTESIA_API_KEY=${CARTESIA_API_KEY}
      - CARTESIA_VOICE_ID=${CARTESIA_VOICE_ID}
      - FIREBASE_SERVICE_ACCOUNT_PATH=/app/service.json
      # LiveKit connection
      - LIVEKIT_URL=${LIVEKIT_URL}
      - LIVEKIT_API_KEY=${LIVEKIT_API_KEY}
      - LIVEKIT_API_SECRET=${LIVEKIT_API_SECRET}
    networks:
      - restaurant-net
    depends_on:
      - livekit
      - freeswitch
    command: python main.py

  # Menu management service
  menu-seeder:
    build: .
    volumes:
      - .:/app
      - ./service.json:/app/service.json
    environment:
      - FIREBASE_SERVICE_ACCOUNT_PATH=/app/service.json
    networks:
      - restaurant-net
    profiles:
      - tools
    command: python scripts/quick_seed.py

networks:
  restaurant-net:
    driver: bridge
Great! now for the best part, building our data persistence. We need a menu in place, and where we will store all customer information, Firestore fast read and write capability will enhance the way we store the call details.

Long Term Memory: Firebase
Create a Firestore manager, remember the key you generated and downloaded, you will need to paste it into the directory and will be used to initialize firebase,

# manager.py

class FirebaseManager:
    """Handles all Firebase operations for the restaurant"""
    
    def __init__(self, credentials_path: str):
        """Initialize Firebase Admin SDK"""
        try:
            # Initialize Firebase Admin
            cred = credentials.Certificate(credentials_path)
            firebase_admin.initialize_app(cred)
            
            # Get Firestore client
            self.db = firestore.client()
            
            # Collection references
            self.menu_collection = self.db.collection('menu_items')
            self.orders_collection = self.db.collection('orders')
            self.customers_collection = self.db.collection('customers')
            self.call_logs_collection = self.db.collection('call_logs')
            self.analytics_collection = self.db.collection('analytics')
            
            # In-memory menu cache for faster access
            self.menu_items = {}
            
              # Load menu items into memory asynchronously
            asyncio.create_task(self._load_menu())
            
            logger.info("Firebase initialized successfully")
        except Exception as e:
            logger.error(f"Firebase initialization failed: {e}")
            raise

    
    async def _load_menu(self):
        """Load menu items into memory for faster access"""
        try:
            docs = self.menu_collection.where('available', '==', True).stream()
            self.menu_items = {doc.id: doc.to_dict() for doc in docs}
            logger.info(f"Loaded {len(self.menu_items)} menu items into memory")
        except Exception as e:
            logger.error(f"Error loading menu: {e}")
You can use a script to initialize the menu which is how I did it, we are using the menu_items cache to ensure that everything is fast access.

Since its a restaurant we need a way to handle the reservations and order, so add the following functions in. We have also to add and/or retrieve customer information so we can relate the order/reservation to the right person.

def save_reservation(self, userdata: UserData) -> bool:
    """Save reservation to Firebase"""
    try:
        reservation_data = {
            'id': str(uuid.uuid4()),
            'customer_name': userdata.customer_name,
            'customer_phone': userdata.customer_phone,
            'reservation_date': userdata.reservation_date,
            'reservation_time': userdata.reservation_time,
            'party_size': userdata.party_size,
            'created_at': datetime.now(),
            'status': 'confirmed',
            'call_id': userdata.call_id
        }
        
        self.db.collection('reservations').add(reservation_data)
        logger.info(f"Saved reservation for {userdata.customer_name}")
        return True
        
    except Exception as e:
        logger.error(f"Error saving reservation: {e}")
        return False


def create_order(self, order: CustomerOrder) -> bool:
      """Create new order in Firebase"""
      try:
          order_ref = self.orders_collection.document(order.id)
          order_ref.set(order.to_dict())
          
          # Update customer info
          self._update_customer_info(order.customer_phone, order.customer_name)
          
          # Update analytics
          self._update_order_analytics(order)
          
          logger.info(f"Created order {order.id} for ${order.total_amount}")
          return True
          
      except Exception as e:
          logger.error(f"Error creating order: {e}")
          return False

def _update_order_analytics(self, order: CustomerOrder):
        """Update daily analytics with new order"""
        try:
            today = datetime.now().date()
            analytics_ref = self.analytics_collection.document(today.isoformat())
            
            analytics_ref.set({
                'date': today,
                'total_orders': firestore.Increment(1),
                'total_revenue': firestore.Increment(order.total_amount),
                'average_order_value': firestore.Increment(order.total_amount),
                'updated_at': datetime.now()
            }, merge=True)
            
        except Exception as e:
            logger.error(f"Error updating analytics: {e}")

# === CUSTOMER OPERATIONS ===
    def _update_customer_info(self, phone: str, name: str):
        """Update or create customer record"""
        try:
            customer_ref = self.customers_collection.document(phone)
            
            # Check if customer exists
            customer_doc = customer_ref.get()
            
            if customer_doc.exists:
                # Update existing customer
                customer_ref.update({
                    'name': name,
                    'last_order_time': datetime.now(),
                    'total_orders': firestore.Increment(1)
                })
            else:
                # Create new customer
                customer_ref.set({
                    'phone': phone,
                    'name': name,
                    'first_order_time': datetime.now(),
                    'last_order_time': datetime.now(),
                    'total_orders': 1,
                    'preferred_items': [],
                    'dietary_restrictions': []
                })
            
        except Exception as e:
            logger.error(f"Error updating customer info: {e}")
Okay great, now we need to set up agents that will be able to save the customers data when they make an order or set up a reservation over the call.

So we need to set up our agents, to handle this. Usually when setting up the agents you need to have a flow of events that would happen in a typical phone call. So the idea is after a call you need to know the intent, are they asking for information, are they making an order are they making a complaint? We need an agent that can help decide this, an IntentClassifierAgent. This agent will be the first one to take the call and decide what the caller wants it will also make greetings.

Short Term Memory: Agent State
But our classifier agent, needs to be able to keep state. That is to be able to allow the data to persist about the user, and the intent so that the many turns won’t be lost and end up in a cycle in the call trying to decipher which action to take. Livekit allows for memory cache using RunContext. And we can use it to define the base state

# models.py

@dataclass
class UserData:
    """Comprehensive user session data"""
    # Customer information
    customer_name: str = ""
    customer_phone: str = ""
    customer_email: str = ""

    # Order management
    current_order: List[Dict] = field(default_factory=list)
    order_total: float = 0.0
    special_instructions: str = ""

    # Reservation management
    reservation_date: str = ""
    reservation_time: str = ""
    party_size: int = 0

    # Call context
    call_id: str = ""
    caller_number: str = ""
    call_start_time: datetime = field(default_factory=datetime.now)
    intent: str = ""
Okay now we use it, pass it to our constructor. To create a DRY flow, we set up a base agent that handles the base calls. It will handle the greetings and handle dependencies, validation, LLM, Set up configuration keys.

class BaseRestaurantAgent(Agent):
    """Base agent that knows how to update and check user data"""

    def __init__(self, chat_ctx: ChatContext, firebase: FirebaseManager, userdata: UserData, config: AgentConfig):
        self.firebase = firebase
        self.userdata = userdata
        self.config = config
        
        # Configure agent with STT, LLM, TTS, VAD and shared tools
        super().__init__(
            chat_ctx=chat_ctx,
            stt=deepgram.STT(api_key=config.deepgram_api_key),
            llm=openai.LLM(
                model=config.openai_model,
                api_key=config.openai_api_key
            ),
            tts=cartesia.TTS(
                api_key=config.cartesia_api_key,
                voice=config.cartesia_voice_id
            ),
            vad=silero.VAD.load(),
            tools=[
                update_customer_name,
                update_customer_phone, 
                update_customer_email,
                add_special_instructions,
                get_customer_summary,
                check_loyalty_status
            ]
        )

    async def on_enter(self):
        """Called when agent becomes active - manage context and greet user"""
        agent_name = self.__class__.__name__
        logger.info(f"Entering {agent_name}")

        userdata: UserData = self.session.userdata
        if userdata.ctx and userdata.ctx.room:
            await userdata.ctx.room.local_participant.set_attributes({"agent": agent_name})

        chat_ctx = self.chat_ctx.copy()

        # Merge context from previous agent if exists
        if hasattr(userdata, 'prev_agent') and userdata.prev_agent:
            items_copy = self._truncate_chat_ctx(
                userdata.prev_agent.chat_ctx.items, keep_function_call=True
            )
            existing_ids = {item.id for item in chat_ctx.items}
            items_copy = [item for item in items_copy if item.id not in existing_ids]
            chat_ctx.items.extend(items_copy)

        # Add system message with current context
        chat_ctx.add_message(
            role="system",
            content=f"You are the {agent_name}. {self.instructions} Current session: {userdata.summarize()}"
        )
        await self.update_chat_ctx(chat_ctx)
        
        # Generate appropriate greeting
        await self.session.generate_reply(
            instructions="Greet the user warmly based on your role and current context.",
        )


    async def _transfer_to_agent(self, name: str, context: RunContext) -> Agent:
        """Transfer to another agent while preserving context"""
        userdata = context.userdata
        current_agent = context.session.current_agent
        if hasattr(userdata, 'personas') and name in userdata.personas:
            next_agent = userdata.personas[name]
            userdata.prev_agent = current_agent
            return next_agent
        else:
            logger.error(f"Agent {name} not found in personas")
            return current_agent

    async def on_exit(self):
        """Called before agent handoff - say goodbye"""
        # Generate appropriate transition message
        await self.session.generate_reply(
            instructions="Let the user know you're transferring them to a specialist who can better help them.",
        )

    async def on_user_turn_completed(self, turn_ctx: ChatContext, new_message: ChatMessage) -> None:
        """Called after user's turn - fetch relevant information from Firebase and enrich context"""
        message_text = new_message.text_content()
        
        # Look up customer information from Firebase based on user's message
        customer_info = await self._lookup_customer_info(message_text)
        
        if customer_info:
            turn_ctx.add_message(
                role="assistant", 
                content=f"Customer information found: {customer_info}"
            )
            # Persist the enriched context
            await self.update_chat_ctx(turn_ctx)

    async def _lookup_customer_info(self, message_text: str) -> str:
        """Look up customer information from Firebase based on message content"""
        import re
        
        # Extract phone numbers from message
        phone_matches = re.findall(r'(\+?1?[-.\s]?\(?[0-9]{3}\)?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4})', message_text)
        
        # Extract email addresses from message
        email_matches = re.findall(r'([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})', message_text)
        
        info_parts = []
        
        # Look up by phone number
        if phone_matches:
            try:
                phone = phone_matches[0]
                customer_history = await self.firebase.get_customer_order_history(phone)
                if customer_history:
                    info_parts.append(f"Found {len(customer_history)} previous orders for {phone}")
                    # Get customer preferences
                    preferences = await self.firebase.get_customer_preferences(phone)
                    if preferences:
                        info_parts.append(f"Customer preferences: {preferences}")
            except Exception as e:
                logger.warning(f"Failed to lookup customer by phone: {e}")
        
        # Look up by email
        if email_matches:
            try:
                email = email_matches[0]
                # You could add email-based lookup here if your Firebase supports it
                info_parts.append(f"Email provided: {email}")
            except Exception as e:
                logger.warning(f"Failed to lookup customer by email: {e}")
        
        return " | ".join(info_parts) if info_parts else ""

    async def _validate_phone(self, phone: str) -> bool:
        """Validate phone number format"""
        import re
        # Simple phone validation - adjust pattern as needed
        phone_pattern = r'^\+?1?[-.\s]?\(?[0-9]{3}\)?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}$'
        return bool(re.match(phone_pattern, phone.strip()))

    async def _validate_email(self, email: str) -> bool:
        """Validate email format"""
        import re
        email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(email_pattern, email.strip()))

    def _handoff_if_done(self):
        # Default: no handoff, override in child if needed
        return None
we apply the base agent so that each specific agent only handles its specific task.

class IntentClassifierAgent(BaseRestaurantAgent):
    def __init__(self, chat_ctx, firebase, userdata: UserData, config: AgentConfig):
        super().__init__(chat_ctx, firebase, userdata, config)
        
        # Get menu information from Firebase
        menu_text = firebase.get_menu_text()
        
        self.instructions = (
            f"You're a friendly cafe staff member. Listen to what the customer wants and help them "
            f"in a warm, natural way. If they want to order food, make a reservation, or just ask questions, "
            f"respond conversationally like you're having a real conversation with a regular customer.\n\n"
            f"Our Menu Today:\n{menu_text}\n\n"
            f"GUIDELINES:\n"
            f"- If they want to order food → use intent_is_order\n"
            f"- If they want to make a reservation → use intent_is_reservation\n"
            f"- Answer simple questions about menu/hours yourself\n"
            f"- Use the shared tools to capture customer information\n"
            f"- Be warm and welcoming, especially to returning customers"
        )
        
    @function_tool()
    async def intent_is_order(self, context: RunContext[UserData]):
        context.userdata.intent = "order"
        return OrderAgent(chat_ctx=self.session._chat_ctx, firebase=self.firebase, userdata=context.userdata, config=self.config)

    @function_tool()
    async def intent_is_reservation(self, context: RunContext[UserData]):
        context.userdata.intent = "reservation"
        return ReservationAgent(chat_ctx=self.session._chat_ctx, firebase=self.firebase, userdata=context.userdata, config=self.config)
With the help of the instructions, its possible to determine which tool to call, and thereby how to route to the next agent.

Here are the OrderAgent for orders, for reservations we use ReservationAgent and to ensure we captured the right details the ConfirmationAgent.

class OrderAgent(BaseRestaurantAgent):
    def __init__(self, chat_ctx, firebase, userdata: UserData, config: AgentConfig):
        super().__init__(chat_ctx, firebase, userdata, config)
        
        # Get menu information from Firebase
        menu_text = firebase.get_menu_text()
        
        self.instructions = (
            f"You're a friendly cafe staff member taking orders. Chat naturally with customers about "
            f"what they'd like to eat or drink. If you need their name or phone for the order, ask casually. "
            f"Only ask for information you actually need - don't interrogate them. Be conversational and helpful, "
            f"like you're talking to a neighbor who stopped by for coffee.\n\n"
            f"Our Menu Today:\n{menu_text}\n\n"
            f"GUIDELINES:\n"
            f"- Help customers choose from our available menu items\n"
            f"- Ask about quantity and any modifications\n"
            f"- Use shared tools to collect customer info (name, phone)\n"
            f"- When order is complete, use finalize_order to proceed to confirmation\n"
            f"- Suggest popular items if customers seem unsure\n"
            f"- Mention loyalty points for returning customers"
        )

    @function_tool()
    async def add_item(self, context: RunContext[UserData], item: str, quantity: int):
        context.userdata.current_order.append({"item": item, "quantity": quantity})
        return self._handoff_if_done()

    @function_tool()
    async def set_payment_method(self, context: RunContext[UserData], method: str):
        context.userdata.payment_method = method
        return self._handoff_if_done()

    @function_tool()
    async def finalize_order(self, context: RunContext[UserData]):
        """Hand off to ConfirmationAgent to review order details"""
        return ConfirmationAgent(
            chat_ctx=self.session._chat_ctx,
            firebase=self.firebase,
            userdata=context.userdata,
            config=self.config
        )

    def _handoff_if_done(self):
        """Check if enough info is collected"""
        if self.userdata.customer_name and self.userdata.customer_phone and self.userdata.current_order:
            return "You may now confirm the order."
        return None

class ReservationAgent(BaseRestaurantAgent):
    def __init__(self, chat_ctx, firebase, userdata: UserData, config: AgentConfig):
        super().__init__(chat_ctx, firebase, userdata, config)
        self.instructions = (
            "You're a friendly cafe staff member helping with table reservations. Chat naturally about "
            "when they'd like to come in and how many people. Ask for their name and phone in a casual way "
            "when you need it. Be warm and welcoming, like you're genuinely excited to have them visit."
        )

    @function_tool()
    async def set_reservation_details(self, context: RunContext[UserData], date: str, time: str, party_size: int):
        context.userdata.reservation_date = date
        context.userdata.reservation_time = time
        context.userdata.party_size = party_size
        return self._handoff_if_done()

    @function_tool()
    async def confirm_reservation(self, context: RunContext[UserData]):
        """Hand off to ConfirmationAgent to review reservation details"""
        return ConfirmationAgent(chat_ctx=self.session._chat_ctx, firebase=self.firebase, userdata=context.userdata, config=self.config)

    def _handoff_if_done(self):
        if (
            self.userdata.customer_name
            and self.userdata.customer_phone
            and self.userdata.reservation_date
            and self.userdata.reservation_time
            and self.userdata.party_size > 0
        ):
            return "You can now use `confirm_reservation` to save this."
        return None


class ConfirmationAgent(BaseRestaurantAgent):
    def __init__(self, chat_ctx, firebase, userdata: UserData, config: AgentConfig):
        super().__init__(chat_ctx, firebase, userdata, config)
        self.instructions = (
            "You're a friendly cafe staff member double-checking the order or reservation. "
            "Be conversational and natural - like you're just making sure you got everything right. "
            "Only mention the key details that matter (what they ordered, when they're coming, etc.). "
            "Don't repeat obvious information. Sound like a real person, not a robot reading a checklist."
        )

    async def on_enter(self):
        """Present key information for confirmation in a natural way"""
        # Create a natural summary focusing on key details
        if hasattr(self.userdata, 'current_order') and self.userdata.current_order:
            # For orders - focus on items and pickup details
            order_summary = self._create_natural_order_summary()
            await self.session.generate_reply(
                instructions=f"Casually confirm the order details: {order_summary}. "
                "Sound natural and friendly, like you're just double-checking you got their order right.",
            )
        elif hasattr(self.userdata, 'reservation_date') and self.userdata.reservation_date:
            # For reservations - focus on time and party size
            reservation_summary = self._create_natural_reservation_summary()
            await self.session.generate_reply(
                instructions=f"Casually confirm the reservation: {reservation_summary}. "
                "Sound warm and welcoming, like you're looking forward to seeing them.",
            )
        else:
            await self.session.generate_reply(
                instructions="Something seems to be missing. Let me help you complete your order or reservation.",
            )

    def _create_natural_order_summary(self) -> str:
        """Create a natural summary of the order focusing on key details"""
        items = []
        if hasattr(self.userdata, 'current_order') and self.userdata.current_order:
            for item in self.userdata.current_order:
                qty = item.get('quantity', 1)
                name = item.get('item', '')
                if qty > 1:
                    items.append(f"{qty} {name}")
                else:
                    items.append(name)
        
        order_text = ", ".join(items) if items else "your order"
        
        # Add name if we have it
        name_part = ""
        if hasattr(self.userdata, 'customer_name') and self.userdata.customer_name:
            name_part = f" for {self.userdata.customer_name}"
        
        return f"{order_text}{name_part}"

    def _create_natural_reservation_summary(self) -> str:
        """Create a natural summary of the reservation focusing on key details"""
        parts = []
        
        if hasattr(self.userdata, 'reservation_date') and self.userdata.reservation_date:
            parts.append(f"on {self.userdata.reservation_date}")
        
        if hasattr(self.userdata, 'reservation_time') and self.userdata.reservation_time:
            parts.append(f"at {self.userdata.reservation_time}")
        
        if hasattr(self.userdata, 'party_size') and self.userdata.party_size:
            people = "person" if self.userdata.party_size == 1 else "people"
            parts.append(f"for {self.userdata.party_size} {people}")
        
        if hasattr(self.userdata, 'customer_name') and self.userdata.customer_name:
            parts.append(f"under {self.userdata.customer_name}")
        
        return "table " + " ".join(parts) if parts else "your reservation"

    @function_tool()
    async def confirm_order(self, context: RunContext[UserData]):
        """Confirm and save the order to Firebase"""
        try:
            if not (context.userdata.customer_name and context.userdata.customer_phone and 
                    hasattr(context.userdata, 'current_order') and context.userdata.current_order):
                return "I just need to get your name and phone number to complete this order."
            
            from app.models.restaurant import CustomerOrder, OrderItem
            from datetime import datetime
            import uuid
            
            # Convert current_order items to OrderItem objects
            order_items = []
            for item_data in context.userdata.current_order:
                order_item = OrderItem(
                    menu_item_id=item_data.get('item', ''),
                    quantity=item_data.get('quantity', 1),
                    unit_price=0.0,  # You'll need to look up the actual price
                    modifications=item_data.get('modifications', [])
                )
                order_items.append(order_item)
            
            # Create CustomerOrder object
            order = CustomerOrder(
                id=str(uuid.uuid4()),
                customer_phone=context.userdata.customer_phone,
                customer_name=context.userdata.customer_name,
                items=order_items,
                order_time=datetime.now(),
                status='pending',
                payment_method=getattr(context.userdata, 'payment_method', 'cash'),
                call_id=getattr(context.userdata, 'call_id', ''),
                agent_id=getattr(context.userdata, 'agent_id', ''),
                special_instructions=getattr(context.userdata, 'special_instructions', '')
            )
            
            await self.firebase.create_order(order)
            return "Perfect! Your order is all set. We'll have it ready for you soon!"
            
        except Exception as e:
            logger.error(f"Failed to save order: {e}")
            return "Oops, something went wrong on our end. Let me try that again for you."

    @function_tool()
    async def confirm_reservation(self, context: RunContext[UserData]):
        """Confirm and save the reservation to Firebase"""
        try:
            if not (context.userdata.customer_name and context.userdata.customer_phone and 
                    hasattr(context.userdata, 'reservation_date') and context.userdata.reservation_date and
                    hasattr(context.userdata, 'reservation_time') and context.userdata.reservation_time and
                    hasattr(context.userdata, 'party_size') and context.userdata.party_size > 0):
                return "I just need a few more details to get your table reserved."
            
            await self.firebase.save_reservation(context.userdata)
            return "Great! Your table is reserved. We're looking forward to seeing you!"
            
        except Exception as e:
            logger.error(f"Failed to save reservation: {e}")
            return "Oops, something went wrong on our end. Let me try that again for you."

    @function_tool()
    async def request_correction(self, context: RunContext[UserData], field: str, new_value: str):
        """Allow customer to correct specific information"""
        field_mapping = {
            'name': 'customer_name',
            'phone': 'customer_phone', 
            'email': 'customer_email',
            'date': 'reservation_date',
            'time': 'reservation_time',
            'party_size': 'party_size'
        }
        
        if field in field_mapping:
            attr_name = field_mapping[field]
            if field == 'party_size':
                setattr(context.userdata, attr_name, int(new_value))
            else:
                setattr(context.userdata, attr_name, new_value)
            return f"Got it, changed to {new_value}. Does everything look right now?"
        
        return "Just let me know what you'd like to change - your name, phone, date, time, or party size."

    @function_tool()
    async def cancel_request(self) -> str:
        """Cancel the current request"""
        return "No problem! What else can I help you with?"
The confirmation agent is added in to ensure the details captured are correct and we don’t end up with wrong details. The LLM will ensure it’s natural and less repetitive. You will also notice we are using the confirmation agent to save to firebase everything else saves to the session, to reduce on latency problems caused by database saves and the rest.

Configuration
Now that the agents are set up, we need to add it in to the entrypoint where we also define our config. We load the sensitive keys using a config. Instead of loading everything one by one.

import os
from dataclasses import dataclass
from typing import Optional
import logging

logger = logging.getLogger(__name__)

@dataclass
class AgentConfig:
    """Configuration for agent STT, LLM, TTS, and VAD settings"""
    # Required fields (no defaults) - must come first
    openai_api_key: str
    deepgram_api_key: str
    cartesia_api_key: str
    
    # Optional fields with defaults - must come after required fields
    openai_model: str = "gpt-4o-mini"
    cartesia_voice_id: Optional[str] = None
    vad_model: str = "silero"
    firebase_service_account_path: str = "service.json"
    sip_phone_number: Optional[str] = None
    sip_trunk_address: Optional[str] = None
    sip_username: Optional[str] = None
    sip_password: Optional[str] = None
    sip_outbound_trunk_id: Optional[str] = None

def load_config() -> AgentConfig:
    """Load configuration from environment variables"""
    
    # Required API keys
    openai_api_key = os.getenv("OPENAI_API_KEY")
    deepgram_api_key = os.getenv("DEEPGRAM_API_KEY") 
    cartesia_api_key = os.getenv("CARTESIA_API_KEY")
    
    if not openai_api_key:
        raise ValueError("OPENAI_API_KEY environment variable is required")
    if not deepgram_api_key:
        raise ValueError("DEEPGRAM_API_KEY environment variable is required")
    if not cartesia_api_key:
        raise ValueError("CARTESIA_API_KEY environment variable is required")
    
    # Optional configurations with defaults
    openai_model = os.getenv("OPENAI_MODEL", "gpt-4o-mini")
    cartesia_voice_id = os.getenv("CARTESIA_VOICE_ID")
    firebase_service_account = os.getenv("FIREBASE_SERVICE_ACCOUNT_PATH", "service.json")
    
    # SIP configurations
    sip_phone_number = os.getenv("SIP_PHONE_NUMBER")
    sip_trunk_address = os.getenv("SIP_TRUNK_ADDRESS")
    sip_username = os.getenv("SIP_USERNAME")
    sip_password = os.getenv("SIP_PASSWORD")
    sip_outbound_trunk_id = os.getenv("SIP_OUTBOUND_TRUNK_ID")
    
    logger.info(f"Loaded config - OpenAI model: {openai_model}")
    
    return AgentConfig(
        openai_api_key=openai_api_key,
        openai_model=openai_model,
        deepgram_api_key=deepgram_api_key,
        cartesia_api_key=cartesia_api_key,
        cartesia_voice_id=cartesia_voice_id,
        firebase_service_account_path=firebase_service_account,
        sip_phone_number=sip_phone_number,
        sip_trunk_address=sip_trunk_address,
        sip_username=sip_username,
        sip_password=sip_password,
        sip_outbound_trunk_id=sip_outbound_trunk_id
    )

def validate_config(config: AgentConfig) -> bool:
    """Validate that all required configuration is present"""
    required_fields = [
        config.openai_api_key,
        config.deepgram_api_key, 
        config.cartesia_api_key
    ]
    
    if not all(required_fields):
        logger.error("Missing required API keys in configuration")
        return False
        
    logger.info("Configuration validation passed")
    return True
We also load our firebase manager and add in the service key path.

# Initialize Firebase manager
 firebase = FirebaseManager(config.firebase_service_account_path)
After this, we load SIP configuration which holds the phone number and the details we want. We will talk about SIP in details later on

sip_handler = SipCallHandler(firebase)
        
# Set up SIP trunk on first run
await sip_handler.setup_sip_trunk(config)

# Create call info (SIP info automatically handled by LiveKit when sip_enabled=True)
call_info = {
    "call_id": ctx.room.name,
    "caller_number": getattr(ctx.job, 'sip', {}).get('from_number', 'unknown') if hasattr(ctx.job, 'sip') else "web_user",
    "called_number": getattr(ctx.job, 'sip', {}).get('to_number', 'restaurant') if hasattr(ctx.job, 'sip') else "restaurant",
    "call_type": "sip_call" if hasattr(ctx.job, 'sip') and ctx.job.sip else "web_session"
}
We also initialize our user data ensuring its unique for every conversation by adding in the room id.

# Create UserData instance for the session
userdata = UserData(
    call_id=call_info["call_id"],
    caller_number=call_info["caller_number"],
    agent_id="restaurant-assistant",
    session_id=call_info["call_id"],
    intent="incoming_call"
)
Once this is complete we can set up the agents, since we have a intent agent to determine what the caller wants, we can initialize the IntentClassifier agent and it will route the caller to the right agent. After that we add in the AgentSession, to coordinate the agents and maintain state. And finally initialize the session.

intent_classifier = IntentClassifierAgent(
    chat_ctx=None,  # Will be set by session
    firebase=firebase,
    userdata=userdata,
    config=config
)

# Start the agent session
session = AgentSession[UserData](userdata=userdata)

await ctx.connect(auto_subscribe=AutoSubscribe.AUDIO_ONLY)

# Start the agent session with error handling
try:
    await session.start(
        agent=intent_classifier,  # Start with the intent classifier
        room=ctx.room,
    )
    
    
except Exception as e:
    # Handle agent session errors
    logger.error(f"Agent session error: {e}")
    await error_handler.handle_agent_error(e, session.run_context, intent_classifier)
    raise
Finally with the entry point complete, we send it to our worker

if __name__ == "__main__":
    # Configure CLI options for SIP support
    cli.run_app(WorkerOptions(
        entrypoint_fnc=entrypoint,
        load_fnc=calculate_worker_load,
        num_idle_processes=2,  # Keep 2 processes warm
        load_threshold=0.8,  # Mark as unavailable at 80% load
        job_memory_warn_mb=1000,  # Warn at 1GB memory usage
        job_memory_limit_mb=2000,  
        max_retry=3,
        port=8081,
    ))
FreeSwitch Configuration
Now we have the app set up and is able to run without any issues and records the data to firebase, but the idea was to explain how to connect SIP to Livekit, and deploy it to Google Cloud. To ensure there are no issues on production we need to set up a local freeswitch server to connect to.

So what is FreeSwitch you might ask.

FreeSwitch is the software that allows a caller to connect to Livekit agents over the internet using your phone and not via the web. In other words its the bridge between the traditional SIP with modern WebRTC which LiveKit is.

To be able to handle this we need to tell Freeswitch how to connect to our Livekit server and not connect to any other server. We will set up the config using the old way, XML.

It uses the vars.xml which acts as keys store and the freeswitch.xml which houses all settings. The vars.xml which will get its values securely from the .env

<?xml version="1.0"?>
<include>
  <!-- FreeSWITCH Variables for Restaurant AI -->
  
  <!-- Network settings -->
  <X-PRE-PROCESS cmd="set" data="default_password=$${FREESWITCH_DEFAULT_PASSWORD}"/>
  <X-PRE-PROCESS cmd="set" data="domain=$${FREESWITCH_DOMAIN}"/>
  <X-PRE-PROCESS cmd="set" data="domain_name=$${FREESWITCH_DOMAIN}"/>
  <X-PRE-PROCESS cmd="set" data="hold_music=$${FREESWITCH_HOLD_MUSIC}"/>
  
  <!-- SIP settings -->
  <X-PRE-PROCESS cmd="set" data="external_rtp_ip=auto-nat"/>
  <X-PRE-PROCESS cmd="set" data="external_sip_ip=auto-nat"/>
  
  <!-- Codec preferences -->
  <X-PRE-PROCESS cmd="set" data="global_codec_prefs=G722,opus@48000h@20i,PCMA,PCMU,GSM"/>
  <X-PRE-PROCESS cmd="set" data="outbound_codec_prefs=G722,opus@48000h@20i,PCMA,PCMU,GSM"/>
  
  <!-- Restaurant specific settings -->
  <X-PRE-PROCESS cmd="set" data="restaurant_name=$${RESTAURANT_NAME}"/>
  <X-PRE-PROCESS cmd="set" data="livekit_url=$${LIVEKIT_URL}"/>
  <X-PRE-PROCESS cmd="set" data="livekit_api_key=$${LIVEKIT_API_KEY}"/>
  <X-PRE-PROCESS cmd="set" data="livekit_secret=$${LIVEKIT_API_SECRET}"/>
  
</include>
The settings

<?xml version="1.0"?>
<document type="freeswitch/xml">
  
  <section name="configuration" description="Various Configuration">
    
    <!-- Event Socket (for LiveKit integration) -->
    <configuration name="event_socket.conf" description="Socket Client">
      <settings>
        <param name="nat-map" value="false"/>
        <param name="listen-ip" value="0.0.0.0"/>
        <param name="listen-port" value="8021"/>
        <param name="password" value="ClueCon"/>
        <param name="apply-inbound-acl" value="loopback.auto"/>
        <param name="stop-on-bind-error" value="true"/>
      </settings>
    </configuration>

    <!-- SIP Profile for incoming calls -->
    <configuration name="sofia.conf" description="sofia Endpoint">
      <global_settings>
        <param name="log-level" value="0"/>
        <param name="abort-on-empty-external-ip" value="true"/>
        <param name="auto-restart" value="false"/>
      </global_settings>
      
      <profiles>
        <profile name="external">
          <gateways>
            <!-- Add your SIP provider gateway here -->
            <gateway name="restaurant-sip">
              <param name="username" value="your-sip-username"/>
              <param name="password" value="your-sip-password"/>
              <param name="realm" value="your-sip-provider.com"/>
              <param name="proxy" value="your-sip-provider.com"/>
              <param name="register" value="true"/>
              <param name="expire-seconds" value="600"/>
              <param name="retry-seconds" value="30"/>
            </gateway>
          </gateways>
          
          <settings>
            <param name="context" value="public"/>
            <param name="rfc2833-pt" value="101"/>
            <param name="sip-port" value="5060"/>
            <param name="dialplan" value="XML"/>
            <param name="dtmf-duration" value="2000"/>
            <param name="codec-prefs" value="PCMU,PCMA"/>
            <param name="use-rtp-timer" value="true"/>
            <param name="rtp-timer-name" value="soft"/>
            <param name="manage-presence" value="false"/>
            <param name="inbound-codec-negotiation" value="generous"/>
            <param name="nonce-ttl" value="60"/>
            <param name="auth-calls" value="false"/>
            <param name="auth-all-packets" value="false"/>
            <param name="ext-rtp-ip" value="auto-nat"/>
            <param name="ext-sip-ip" value="auto-nat"/>
            <param name="rtp-timeout-sec" value="300"/>
            <param name="rtp-hold-timeout-sec" value="1800"/>
            <param name="tls" value="false"/>
            <param name="inbound-late-negotiation" value="true"/>
            <param name="inbound-zrtp-passthru" value="true"/>
          </settings>
        </profile>
      </profiles>
    </configuration>

    <!-- Modules to load -->
    <configuration name="modules.conf" description="Modules">
      <modules>
        <!-- Core modules -->
        <load module="mod_console"/>
        <load module="mod_logfile"/>
        <load module="mod_enum"/>
        <load module="mod_cdr_csv"/>
        <load module="mod_event_socket"/>
        
        <!-- Media/codec modules -->
        <load module="mod_native_file"/>
        <load module="mod_sndfile"/>
        <load module="mod_tone_stream"/>
        <load module="mod_local_stream"/>
        <load module="mod_dptools"/>
        
        <!-- SIP module -->
        <load module="mod_sofia"/>
        
        <!-- Dialplan modules -->
        <load module="mod_dialplan_xml"/>
        
        <!-- Codec modules -->
        <load module="mod_g711"/>
        <load module="mod_g729"/>
        <load module="mod_opus"/> 
        <load module="mod_amr"/>
      </modules>
    </configuration>

  </section>

  <!-- Dialplan for routing calls to LiveKit -->
  <section name="dialplan" description="Regex/XML Dialplan">
    <context name="public">
      
      <!-- Route incoming calls to restaurant assistant -->
      <extension name="restaurant-assistant">
        <condition field="destination_number" expression="^(restaurant|1234567890)$">
          <action application="answer"/>
          <action application="sleep" data="1000"/>
          <action application="set" data="call_direction=inbound"/>
          <action application="set" data="restaurant_call=true"/>
          <!-- This will be handled by LiveKit SIP integration -->
          <action application="socket" data="127.0.0.1:8080 async full"/>
        </condition>
      </extension>
      
      <!-- Default handler -->
      <extension name="default">
        <condition field="destination_number" expression="^.*$">
          <action application="answer"/>
          <action application="playback" data="tone_stream://%(2000,4000,440,480);loops=3"/>
          <action application="hangup"/>
        </condition>
      </extension>
      
    </context>
  </section>

</document>
I will explain each section to show how it will be used to ensure everything works as expected.

So we have two sections, the configuration and the dialplan

<document type="freeswitch/xml">
  <section name="configuration">...</section>
  <section name="dialplan">...</section>
</document>
Configuration: Defines system wide settings
Dialplan: Handles the routing of outgoing and incoming calls and how the call should behave.
Event Socket

<configuration name="event_socket.conf">
  ...
</configuration>
This configuration allows our app to connect to Freeswitch over a TCP connection and control the calls programmatically. Inside it we have the following options

listen-ip=”0.0.0.0"
Listens on all interfaces for socket clients.

2. listen-port=”8021"

Default port for ESL (Event Socket Library) clients.

3. password=”ClueCon”

Default password (change this in production!).

4. apply-inbound-acl=”loopback.auto”

Allows us to test the app locally. You can download the app from playstore to test it out and add in the config details to connect

5. stop-on-bind-error=”true”

Prevents FreeSWITCH from starting if socket can’t bind

SIP Setup
The config sets up the SIP endpoint that FreeSWITCH uses to receive/make calls via SIP.

<configuration name="sofia.conf">
  ...
</configuration>
It is made up of two parts mainly:

global_settings
# Prevents startup errors if the external IP is not detected.
# Its turned off for debugging
<param name="auto-restart" value="false"/>
<param name="abort-on-empty-external-ip" value="true"/>
2. profiles

We are able to define our connection to a SIP like Twilio or we set up our own PBX. Mainly for registration. You will mainly see the following realm, proxy – usually the SIP domain/hostname from our provider.

<gateway name="restaurant-sip">
  ...
</gateway>
Inside the profiles are the settings.

<param name="sip-port" value="5060"/>
<param name="codec-prefs" value="$${global_codec_prefs}"/>
<param name="context" value="public"/>
The sip-port="5060" → default unencrypted SIP port, this matches with what we set in docker compose. context="public" → all calls to this profile land in the public dialplan context. We also set the audio codecs, codec-prefs → allowed audio codecs (PCMU = G.711 µ-law) actually stands for Pulse Code Modulation µ-law. G.711 is the standard telephony audio. Its latency is very low with a bitrate of 64kbps. Its quality is not HD though but very good for testing.
The problem we might have is that Livekit uses Opus codec while on the SIP side PCMU is used. Freeswitch will handle the transcoding between Opus and PMCU. auth-calls="false" → disable auth for inbound calls. We will enable it when deploying to production. rtp-timeout-sec → number of seconds to wait before we drop the call.

Modules

Tells FreeSwitch which plugins to load. In our config we are using the following plugins.

mod_sofia: The SIP stack. Required for SIP functionality.

mod_event_socket: Enables Event Socket for control/monitoring.

mod_g711, mod_g729, mod_amr: Audio codecs for transcoding.

mod_dialplan_xml: Processes the XML dialplan.

mod_console, mod_logfile: Mainly for logging.

<configuration name="modules.conf">
  ...
</configuration>
The current flow being SIP → media → call routing → external control.

DialPlan: Routing the call

The config defines how the call is routed. And we use extension to define how it flows.

<section name="dialplan">
  <context name="public">...</context>
</section>
Currently there are 2:

restaurant-assistant.

This is a very critical section of the app because it sends the media in this case the voice to our LiveKit server we have built. The flow is that when a call comes in freeswitch via the port 5060, it will check whether the number matches with the number we set in. (You can change that to your test number). Once it matches it will route the call control and media via the socket app to a service listening on port 127.0.0.1:8080 . This will happen back and forth to allow for the voice call to happen.

<extension name="restaurant-assistant">
  <condition field="destination_number" expression="^(restaurant|$${RESTAURANT_PHONE_NUMBER})$">
    ...
    <action application="socket" data="backend:8080 async full"/>
  </condition>
</extension>
It is made up of the following components.

answer: Answers the SIP call

sleep 1000: Pause briefly (allow connection to settle)

set call_direction=inbound: This is an optional field and it sets the call metadata

set restaurant_call=true: Handles the routing logic

socket 127.0.0.1:8080 async full: Hands off call control to our livekit server

default

This section handles all invalid calls and handles them gracefully. It answers sends a fake ring and hangs up after 3 retries.

<extension name="default">
  ...
</extension>
Now a very critical part we forgot is the vars.xml . It makes use of .env variables. We can then change these based on whether we are building locally or in production.

<X-PRE-PROCESS cmd="set" data="livekit_url=$${LIVEKIT_URL}"/>
<X-PRE-PROCESS cmd="set" data="livekit_api_key=$${LIVEKIT_API_KEY}"/>
<X-PRE-PROCESS cmd="set" data="livekit_secret=$${LIVEKIT_API_SECRET}"/>
Testing
To be able to test the locally using Zoiper which will call into our FreeSwitch server and connect to our assistant and mimic a production call to get any bugs that may come up.

Deployment
To connect to SIP you would need to choose a provider either one of these

Twilio
Telnyx
Plivo
You will get the SIP URI, and a phone number after purchasing. You can then associate the phone number with the URI depending on whether its outbound or inbound.

Now with all the configuration set up, it needs to be pushed to production. We have made our app to use environment keys so using secrets is going to be good because we can leverage google secret manager and call it from Cloud run to use in a secure way. Cloud build will be good since we can set up a trigger to make a new build from a PR merge on the main branch. The reason for this is that is will auto scale

As for GKE which allows for a more fine grained control is the best use case for production based assistant that serves different ports to millions of users. I will be writing more on this on the next article

Conclusion
Building agents is getting better with time and definitely easier the problem is how those agents scale in production and how the intergrate with traditional more common based technologies. Knowing how to do this well is extremely crucial for a service that not only works but a service that is almost a copy if not better than traditional service. The only reason why someone will use your app.
FreeSwitch is a great service but there are other alternative like

1. Kamailio/OpenSIPs: Extremely performant due to its lightweight. It needs to be paired with FreeSwitch since it does not handle RTP/media. It can also scaled to thousands of calls concurrently

2. JANUS: Small and provides low latency WebRTC to SIP bridging, though it only supports SIP compared to FreeSwitch.

3. Twilio: This is a fully cloud based service its very performant supports Global telephony but very expensive

You can get the link to the repo here 
