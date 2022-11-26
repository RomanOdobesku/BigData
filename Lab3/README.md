# Lab 3 - Stream processing with Apache Flink
Тесты для каждого упражнения были пройдены. Решения на Java.
## Решение
### RideCleanisingExercise
#### Задание
The task of the exercise is to filter a data stream of taxi ride records to keep only rides that start and end within New York City. The resulting stream should be printed.
#### Код
```java
private static class NYCFilter implements FilterFunction<TaxiRide> {

    @Override
    public boolean filter(TaxiRide taxiRide) throws Exception {
        return GeoUtils.isInNYC(taxiRide.startLon, taxiRide.startLat) && GeoUtils.isInNYC(taxiRide.endLon, taxiRide.endLat);
    }
}
```
#### Пояснение
С помощью функции isInNYC из библиотеки GeoUtils определяем поездки, которые начаты и закончены в Нью-Йорке. Для этого используются координаты - широта и долгота (для начала и конца поездки).
```java
import com.ververica.flinktraining.exercises.datastream_java.utils.GeoUtils;
```
### RidesAndFaresExercise
#### Задание
The goal for this exercise is to enrich TaxiRides with fare information.
#### Код
```java
public static class EnrichmentFunction extends RichCoFlatMapFunction<TaxiRide, TaxiFare, Tuple2<TaxiRide, TaxiFare>> {

  private ValueState<TaxiRide> taxiRideValueState;
  private ValueState<TaxiFare> taxiFareValueState;

  @Override
  public void open(Configuration config) throws Exception {
    ValueStateDescriptor<TaxiRide> taxiRideValueStateDescriptor = new ValueStateDescriptor<TaxiRide>(
        "persistedTaxiRide", TaxiRide.class
    );
    ValueStateDescriptor<TaxiFare> taxiFareValueStateDescriptor = new ValueStateDescriptor<TaxiFare>(
        "persistedTaxiFare", TaxiFare.class
    );

    this.taxiRideValueState = getRuntimeContext().getState(taxiRideValueStateDescriptor);
    this.taxiFareValueState = getRuntimeContext().getState(taxiFareValueStateDescriptor);
  }

  @Override
  public void flatMap1(TaxiRide ride, Collector<Tuple2<TaxiRide, TaxiFare>> out) throws Exception {
    TaxiFare taxiFare = this.taxiFareValueState.value();
    if (taxiFare != null) {
      this.taxiFareValueState.clear();
      out.collect(new Tuple2<>(ride, taxiFare));
    } else {
      this.taxiRideValueState.update(ride);
    }
  }

  @Override
  public void flatMap2(TaxiFare fare, Collector<Tuple2<TaxiRide, TaxiFare>> out) throws Exception {
    TaxiRide taxiRide = this.taxiRideValueState.value();
    if (taxiRide != null) {
      this.taxiRideValueState.clear();
      out.collect(new Tuple2<>(taxiRide, fare));
    } else {
      this.taxiFareValueState.update(fare);
    }
  }
}
```
#### Пояснение
С помощью EnrichmentFunction, наследующейся от RichCoFlatMapFunction будем соединять пары ride-fare по ключу rideId. flatMap1 и flatMap2 принимают на вход TaxiRide или TaxiFare соответственно, если в поле класса содержится значение taxiFare или taxiRide соответственно, то применяется out.collect с переданным набором из 2 элементов, иначе поданная на вход сущность записывается в поле класса.
### HourlyTipsExerxise
#### Задание
The task of the exercise is to first calculate the total tips collected by each driver, hour by hour, and then from that stream, find the highest tip total in each hour.
#### Код
```java
public class HourlyTipsExercise extends ExerciseBase {

	public static void main(String[] args) throws Exception {

		// read parameters
		ParameterTool params = ParameterTool.fromArgs(args);
		final String input = params.get("input", ExerciseBase.pathToFareData);

		final int maxEventDelay = 60;       // events are out of order by max 60 seconds
		final int servingSpeedFactor = 600; // events of 10 minutes are served in 1 second

		// set up streaming execution environment
		StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
		env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
		env.setParallelism(ExerciseBase.parallelism);

		// start the data generator
		DataStream<TaxiFare> fares = env.addSource(fareSourceOrTest(new TaxiFareSource(input, maxEventDelay, servingSpeedFactor)));
		// compute tips per hour for each driver
		DataStream<Tuple3<Long, Long, Float>> hourlyTips =
				fares.keyBy(fare -> fare.driverId)
						.timeWindow(Time.hours(1))
						.process(new CalculateHourlyTips());

		// find the driver with the highest sum of tips for each hour
		DataStream<Tuple3<Long, Long, Float>> hourlyMax =
				hourlyTips.timeWindowAll(Time.hours(1))
						.maxBy(2);


		printOrTest(hourlyMax);

		// execute the transformation pipeline
		env.execute("Hourly Tips (java)");
	}

	private static class CalculateHourlyTips
			extends ProcessWindowFunction<TaxiFare, Tuple3<Long, Long, Float>, Long, TimeWindow> {

		@Override
		public void process(
				Long key,
				Context context,
				Iterable<TaxiFare> fares,
				Collector<Tuple3<Long, Long, Float>> out) {

			float tipsSum = 0.0f;
			for (TaxiFare fare : fares) {
				tipsSum += fare.tip;
			}
			out.collect(Tuple3.of(context.window().getEnd(), key, tipsSum));
		}
	}
}
```
#### Пояснение
Класс CalculateHourlyTips реализует функцию process с помощью которой можно посчитать сумму чаевыех за час для каждого водителя. После этого с помощью hourlyTips.timeWindowAll(Time.hours(1)).maxBy(2) выберем наибольшие чаевые за час и вернем объекты, состоящие из времени конца временного окна, driverId и суммы его чаевых за этот час. 
### ExpiringStateExercise
#### Задание
The goal for this exercise is to enrich TaxiRides with fare information.
#### Код
```java
public static class EnrichmentFunction extends KeyedCoProcessFunction<Long, TaxiRide, TaxiFare, Tuple2<TaxiRide, TaxiFare>> {

  private ValueState<TaxiRide> taxiRideValueState;
  private ValueState<TaxiFare> taxiFareValueState;

  @Override
  public void open(Configuration config) throws Exception {
    ValueStateDescriptor<TaxiRide> taxiRideDescriptor = new ValueStateDescriptor<>(
        "persistedTaxiRide", TaxiRide.class
    );
    ValueStateDescriptor<TaxiFare> taxiFareDescriptor = new ValueStateDescriptor<>(
        "persistedTaxiFare", TaxiFare.class
    );

    this.taxiRideValueState = getRuntimeContext().getState(taxiRideDescriptor);
    this.taxiFareValueState = getRuntimeContext().getState(taxiFareDescriptor);
  }

  @Override
  public void onTimer(long timestamp, OnTimerContext ctx, Collector<Tuple2<TaxiRide, TaxiFare>> out) throws Exception {
    if (this.taxiFareValueState.value() != null) {
      ctx.output(unmatchedFares, this.taxiFareValueState.value());
      this.taxiFareValueState.clear();
    }
    if (this.taxiRideValueState.value() != null) {
      ctx.output(unmatchedRides, this.taxiRideValueState.value());
      this.taxiRideValueState.clear();
    }
  }

  @Override
  public void processElement1(TaxiRide ride, Context context, Collector<Tuple2<TaxiRide, TaxiFare>> out) throws Exception {
    TaxiFare fare = this.taxiFareValueState.value();
    if (fare != null) {
      this.taxiFareValueState.clear();
      context.timerService().deleteEventTimeTimer(ride.getEventTime());
      out.collect(new Tuple2<>(ride, fare));
    } else {
      this.taxiRideValueState.update(ride);
      context.timerService().registerEventTimeTimer(ride.getEventTime());
    }
  }

  @Override
  public void processElement2(TaxiFare fare, Context context, Collector<Tuple2<TaxiRide, TaxiFare>> out) throws Exception {
    TaxiRide ride = this.taxiRideValueState.value();
    if (ride != null) {
      this.taxiRideValueState.clear();
      context.timerService().deleteEventTimeTimer(fare.getEventTime());
      out.collect(new Tuple2<>(ride, fare));
    } else {
      this.taxiFareValueState.update(fare);
      context.timerService().registerEventTimeTimer(fare.getEventTime());
    }
  }
}
```
#### Пояснение
Здесь результатом работы являются объекты, записанные с тегами unmatchedRides и unmatchedFares. Объекты с этими тегами - это объекты rides и fares, которые не смогли объединиться (как в задании 2) за отведенное таймером время. Объединение происходит по rideId, некоторые объекты не могут объединится из-за условия ride.rideId % 1000 != 0 в фильтре rides объектов.
