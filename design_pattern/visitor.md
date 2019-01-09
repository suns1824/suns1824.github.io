```text
public interface City {
  public void accept(Visitor visitor);
}

public class Beijing implements City {
  @Override
  public void accept(Visitor visitor) {
    visitor.visit(this);
  }
}

public class Shanghai implements City {
  @Override
  public void accept(Visitor visitor) {
    visitor.visit(this);
  }
}

public class TravelCities implements City {
  City[] cities;
  public TravelCities() {
    cities = new City[] {new Beijing(), new Shanghai()};
  }
  @Override
  public void accept(Visitor visitor) {
    for(int i = 0; i < cities.length;i++) {
      cities[i].accept(visitor);
    }
    }
  }
}

public interface Visitor {
  public void visit(Beijing beijing);
  public void visit(Shanghai shanghai);
}

public interface SingleVisitor implements Visitor {
  @Override
  public void visit(Beijing beijing) {
    System.out.println("Beijing");
  }
  @Override
  public void visit(Shanghai shanghai) {
    System.out.println("shanghai");
  }
}


```